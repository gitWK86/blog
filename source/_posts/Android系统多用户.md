# Android系统多用户

## 概述

Android从4.2开始支持多用户模式，不同的用户运行在独立的用户空间，它们的的锁屏设置，PIN码和壁纸等系统设置是各不相同的，而且不同用户安装的应用和应用数据都是不一样的，但是系统中和硬件相关的设置则是共用的，比如网络设置等。通常第一个在系统中注册的用户将成为系统管理员，可以管理手机上的其他用户。

手机上经常见到的一个功能就是访客模式，它是多用户模式的一种应用。访客模式下用户的所有数据（通讯录，短信，应用等）会被隐藏，访客只能查看和使用手机的基本功能，另外你还可以设置访客是否有接听电话、发送短信等权限。

## adb相关命令

1. 获取用户列表、

   `pm list users`

2. 创建新用户

   `pm create-user user-name`

3. 启动用户

   `am start-user user-id`

4. 切换用户

   `am switch-user user-id`

5. 获取所有用户信息

   `dumpsys user`

6. 删除用户

   `pm remove-user user-id`

7. 获取最大用户数

   `pm get-max-users`

## UserManagerService

UMS 是用来管理用户的系统服务，是创建、删除以及查询用户的执行者。

首先梳理下UMS初始化流程：

UMS的初始化实在PMS的构造函数中执行的

```java
new UserManagerService(context, pm,new UserDataPreparer(installer, installLock, context, onlyCore), lock)
```

然后看下UMS的构造函数

### 构造函数

`frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java`

```java
UserManagerService(Context context, PackageManagerService pm, UserDataPreparer userDataPreparer,
           Object packagesLock) {
       this(context, pm, userDataPreparer, packagesLock, Environment.getDataDirectory());
}

private UserManagerService(Context context, PackageManagerService pm,
            UserDataPreparer userDataPreparer, Object packagesLock, File dataDir) {
        mContext = context;
        mPm = pm;
        mPackagesLock = packagesLock;
        mHandler = new MainHandler();
        mUserDataPreparer = userDataPreparer;
        mUserTypes = UserTypeFactory.getUserTypes();
        synchronized (mPackagesLock) {
            // 创建/data/system/users目录
            mUsersDir = new File(dataDir, USER_INFO_DIR);
            mUsersDir.mkdirs();
            // Make zeroth user directory, for services to migrate their files to that location
            // 创建第一个用户的目录/data/system/users/0
            File userZeroDir = new File(mUsersDir, String.valueOf(UserHandle.USER_SYSTEM));
            userZeroDir.mkdirs();
            // 设置目录权限
            FileUtils.setPermissions(mUsersDir.toString(),
                    FileUtils.S_IRWXU | FileUtils.S_IRWXG | FileUtils.S_IROTH | FileUtils.S_IXOTH,
                    -1, -1);
            
            // 创建代表/data/system/users/userlist.xml文件的对象
            mUserListFile = new File(mUsersDir, USER_LIST_FILENAME);
            // 添加一些对访客用户的默认限制
            initDefaultGuestRestrictions();
            // 读取userlist.xml文件，将用户的信息保存在mUsers列表中，如果文件不存在就创建一个
            readUserListLP();
            sInstance = this;
        }
        mSystemPackageInstaller = new UserSystemPackageInstaller(this, mUserTypes);
        mLocalService = new LocalService();
        LocalServices.addService(UserManagerInternal.class, mLocalService);
        mLockPatternUtils = new LockPatternUtils(mContext);
        mUserStates.put(UserHandle.USER_SYSTEM, UserState.STATE_BOOTING);
    }
```

在构造函数中，主要的操作就是创建了`/data/system/users`目录，并在该目录下创建`0/`目录。然后分析`/data/system/users/userlist.xml`文件，这个文件保存了系统中所有用户的Id信息。当前状态如下：

![](Android系统多用户\1.png)

`userlist.xml`文件

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<users nextSerialNumber="14" version="9">
    <guestRestrictions>
        <restrictions no_sms="true" no_install_unknown_sources="true" no_config_wifi="true" no_outgoing_calls="true" />
    </guestRestrictions>
    <deviceOwnerUserId id="-10000" />
    <user id="0" />
</users>
```

`nextSerialNumber`指的是创建下一个用户时它的`serialNumber`，`guestRestrictions`标签指的是为访客用户设置的权限。`deviceOwnerUserId`是`userlist.xml`文件创建时写入的一个空用户。`<user id="0" />`，0就表示当前主用户。

`0.xml`文件

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<user id="0" serialNumber="0" flags="3091" type="android.os.usertype.full.SYSTEM" created="0" lastLoggedIn="1625020212911" lastLoggedInFingerprint="google/sdk_gphone_x86_arm/generic_x86_arm:11/RSR1.201013.001/6903271:userdebug/dev-keys" profileBadge="0">
    <restrictions />
    <device_policy_local_restrictions />
</user>
```

标签的属性值对应了`UserInfo`里面的成员变量，读取用户的xml文件后，会根据文件的内容来创建和初始化一个`UserInfo`来保存，并把该对象加入到`mUsers`列表中去。

## 创建新用户

![create时序图](Android系统多用户\pm create-user.png)



新用户的创建逻辑主要是在UserManagerService中处理。关键代码如下:

`frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java`

```java
private UserInfo createUserInternalUncheckedNoTracing(@Nullable String name,
            @NonNull String userType, @UserInfoFlag int flags, @UserIdInt int parentId,
            boolean preCreate, @Nullable String[] disallowedPackages,
            @NonNull TimingsTraceAndSlog t) throws UserManager.CheckedUserOperationException {
        final UserTypeDetails userTypeDetails = mUserTypes.get(userType);
        if (userTypeDetails == null) {
            Slog.e(LOG_TAG, "Cannot create user of invalid user type: " + userType);
            return null;
        }
    
        // 检查是否是低内存设备
        DeviceStorageMonitorInternal dsm = LocalServices
                .getService(DeviceStorageMonitorInternal.class);
        if (dsm.isMemoryLow()) {
            throwCheckedUserOperationException("Cannot add user. Not enough space on disk.",
                    UserManager.USER_OPERATION_ERROR_LOW_STORAGE);
        }

        final boolean isProfile = userTypeDetails.isProfile();
        final boolean isGuest = UserManager.isUserTypeGuest(userType);
        final boolean isRestricted = UserManager.isUserTypeRestricted(userType);
        final boolean isDemo = UserManager.isUserTypeDemo(userType);

        final long ident = Binder.clearCallingIdentity();
        UserInfo userInfo;
        UserData userData;
        final int userId;
        try {
            synchronized (mPackagesLock) {
                UserData parent = null;
                ......

                // 1、为新用户创建userID，userId从10开始递增
                userId = getNextAvailableId();
                // 创建 "/data/system/users/${userId}"目录
                Environment.getUserSystemDirectory(userId).mkdirs();

                synchronized (mUsersLock) {
                  
                    // 2、构造UserInfo和UserData
                    userInfo = new UserInfo(userId, name, null, flags, userType);
                    userInfo.serialNumber = mNextSerialNumber++;
                    userInfo.creationTime = getCreationTime();
                    // 正在创建标致
                    userInfo.partial = true;
                    userInfo.preCreated = preCreate;
                    userInfo.lastLoggedInFingerprint = Build.FINGERPRINT;
                    if (userTypeDetails.hasBadge() && parentId != UserHandle.USER_NULL) {
                        userInfo.profileBadge = getFreeProfileBadgeLU(parentId, userType);
                    }
                    userData = new UserData();
                    userData.info = userInfo;
                    mUsers.put(userId, userData);
                }
                
                //将UserData信息写入到 "/data/system/users/${userId}.xml"
                writeUserLP(userData);
                //将新创建的userId添加到 "/data/system/users/userlist.xml"
                writeUserListLP();
                ......
            }

            t.traceBegin("createUserKey");
            // 3、为新用户准备文件系统
            final StorageManager storage = mContext.getSystemService(StorageManager.class);
            // 通过vold对新用户进行文件系统加密
            storage.createUserKey(userId, userInfo.serialNumber, userInfo.isEphemeral());
            t.traceEnd();

            t.traceBegin("prepareUserData");
             // 通过vold创建以下目录，并赋予相关rwx权限：
             // "/data/system/users/${userId}" : 0700
             // "/data/misc/users/${userId}" : 0750
            mUserDataPreparer.prepareUserData(userId, userInfo.serialNumber,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
            t.traceEnd();

            final Set<String> userTypeInstallablePackages =
                    mSystemPackageInstaller.getInstallablePackagesForUserType(userType);
            t.traceBegin("PM.createNewUser");
            // 4、为已安装应用创建"/data/user/${userId}/${packageName}"目录
            mPm.createNewUser(userId, userTypeInstallablePackages, disallowedPackages);
            t.traceEnd();

            // 5、用户已经创建完成，固化用户创建状态
            userInfo.partial = false;
            synchronized (mPackagesLock) {
                writeUserLP(userData);
            }
            
            // 更新所有缓存的用户
            updateUserIds();

            Bundle restrictions = new Bundle();
            if (isGuest) {
                // Guest default restrictions can be modified via setDefaultGuestRestrictions.
                synchronized (mGuestRestrictions) {
                    restrictions.putAll(mGuestRestrictions);
                }
            } else {
                userTypeDetails.addDefaultRestrictionsTo(restrictions);
            }
            synchronized (mRestrictionsLock) {
                mBaseUserRestrictions.updateRestrictions(userId, restrictions);
            }

            t.traceBegin("PM.onNewUserCreated-" + userId);
            // 6、为新创建的用户赋予默认权限
            mPm.onNewUserCreated(userId);
            t.traceEnd();
        
            // 7、向所有用户发送 "ACTION_USER_ADDED" 广播
            dispatchUserAdded(userInfo);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        return userInfo;
    }
```



在创建用户的流程中，主要做了以下几件事：

1. **为新用户创建一个新的userId**
2. **固化新用户信息和创建状态**
3. **准备文件系统**
4. **为已安装应用准备数据目录并记录其组件和默认权限配置**
5. **固化新用户创建完成的状态**
6. **通知PMS为新用户和应用赋予默认的权限**
7. **发送 “ACTION_USER_ADDED” 广播，新用户创建完成**

创建用户（14）后在`/data/system/users`目录下就会创建14文件夹、14.xml

![](Android系统多用户\2.png)

`14.xml`文件

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<user id="14" serialNumber="14" flags="1024" type="android.os.usertype.full.SECONDARY" created="1627010294107" lastLoggedIn="0" lastLoggedInFingerprint="google/sdk_gphone_x86_arm/generic_x86_arm:11/RSR1.201013.001/6903271:userdebug/dev-keys" profileBadge="0">
    <name>test</name>
    <device_policy_local_restrictions />
</user>

```

`userlist.xml`文件

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<users nextSerialNumber="15" version="9">
    <guestRestrictions>
        <restrictions no_sms="true" no_install_unknown_sources="true" no_config_wifi="true" no_outgoing_calls="true" />
    </guestRestrictions>
    <deviceOwnerUserId id="-10000" />
    <user id="0" />
    <user id="14" />
</users>

```

### 接收ACTION_USER_ADDED广播列表 

| 服务                        |      |
| --------------------------- | ---- |
| StorageManagerService       |      |
| ConnectivityService         |      |
| CameraServiceProxy          |      |
| InputMethodManagerService   |      |
| LockSettingsService         |      |
| NetworkPolicyManagerService |      |
| NotificationManagerService  |      |
| OverlayManagerService       |      |
| RollbackManagerServiceImpl  |      |
| TrustManagerService         |      |
| DevicePolicyManagerService  |      |



## 启动用户

`frameworks/base/services/core/java/com/android/server/am/UserController.java`

```java
 private boolean startUserInternal(@UserIdInt int userId, boolean foreground,
            @Nullable IProgressListener unlockListener, @NonNull TimingsTraceAndSlog t) {
        EventLog.writeEvent(EventLogTags.UC_START_USER_INTERNAL, userId);

        final int callingUid = Binder.getCallingUid();
        final int callingPid = Binder.getCallingPid();
        final long ident = Binder.clearCallingIdentity();
        try {
            t.traceBegin("getStartedUserState");
            final int oldUserId = getCurrentUserId();
            // 如果当前用户已经是需要切换的用户
            if (oldUserId == userId) {
                final UserState state = getStartedUserState(userId);
                ......
                if (state.state == STATE_RUNNING_UNLOCKED) {
                    // We'll skip all later code, so we must tell listener it's already
                    // unlocked.
                    notifyFinished(userId, unlockListener);
                }
                t.traceEnd(); //getStartedUserState
                return true;
            }
            t.traceEnd(); //getStartedUserState

            if (foreground) {
                t.traceBegin("clearAllLockedTasks");
                mInjector.clearAllLockedTasks("startUser");
                t.traceEnd();
            }

            t.traceBegin("getUserInfo");
            final UserInfo userInfo = getUserInfo(userId);
            t.traceEnd();
            
            // 如果没有需要启动的用户的信息，则直接退出
            if (userInfo == null) {
                Slog.w(TAG, "No user info for user #" + userId);
                return false;
            }
            if (foreground && userInfo.isManagedProfile()) {
                Slog.w(TAG, "Cannot switch to User #" + userId + ": not a full user");
                return false;
            }

            if (foreground && userInfo.preCreated) {
                Slog.w(TAG, "Cannot start pre-created user #" + userId + " as foreground");
                return false;
            }

            if (foreground && isUserSwitchUiEnabled()) {
                t.traceBegin("startFreezingScreen");
                mInjector.getWindowManager().startFreezingScreen(
                        R.anim.screen_user_exit, R.anim.screen_user_enter);
                t.traceEnd();
            }

            boolean needStart = false;
            boolean updateUmState = false;
            UserState uss;

            // If the user we are switching to is not currently started, then
            // we need to start it now.
            t.traceBegin("updateStartedUserArrayStarting");
            synchronized (mLock) {
                uss = mStartedUsers.get(userId);
                if (uss == null) {
                    uss = new UserState(UserHandle.of(userId));
                    uss.mUnlockProgress.addListener(new UserProgressListener());
                    mStartedUsers.put(userId, uss);
                    updateStartedUserArrayLU();
                    needStart = true;
                    updateUmState = true;
                } else if (uss.state == UserState.STATE_SHUTDOWN && !isCallingOnHandlerThread()) {
                    Slog.i(TAG, "User #" + userId
                            + " is shutting down - will start after full stop");
                    mHandler.post(() -> startUser(userId, foreground, unlockListener));
                    t.traceEnd(); // updateStartedUserArrayStarting
                    return true;
                }
                final Integer userIdInt = userId;
                mUserLru.remove(userIdInt);
                mUserLru.add(userIdInt);
            }
            if (unlockListener != null) {
                uss.mUnlockProgress.addListener(unlockListener);
            }
            t.traceEnd(); // updateStartedUserArrayStarting

            if (updateUmState) {
                t.traceBegin("setUserState");
                mInjector.getUserManagerInternal().setUserState(userId, uss.state);
                t.traceEnd();
            }
            t.traceBegin("updateConfigurationAndProfileIds");
            if (foreground) {
                // Make sure the old user is no longer considering the display to be on.
                mInjector.reportGlobalUsageEventLocked(UsageEvents.Event.SCREEN_NON_INTERACTIVE);
                boolean userSwitchUiEnabled;
                synchronized (mLock) {
                    mCurrentUserId = userId;
                    mTargetUserId = UserHandle.USER_NULL; // reset, mCurrentUserId has caught up
                    userSwitchUiEnabled = mUserSwitchUiEnabled;
                }
                mInjector.updateUserConfiguration();
                updateCurrentProfileIds();
                mInjector.getWindowManager().setCurrentUser(userId, getCurrentProfileIds());
                mInjector.reportCurWakefulnessUsageEvent();
                // Once the internal notion of the active user has switched, we lock the device
                // with the option to show the user switcher on the keyguard.
                if (userSwitchUiEnabled) {
                    mInjector.getWindowManager().setSwitchingUser(true);
                    mInjector.getWindowManager().lockNow(null);
                }
            } else {
                final Integer currentUserIdInt = mCurrentUserId;
                updateCurrentProfileIds();
                mInjector.getWindowManager().setCurrentProfileIds(getCurrentProfileIds());
                synchronized (mLock) {
                    mUserLru.remove(currentUserIdInt);
                    mUserLru.add(currentUserIdInt);
                }
            }
            t.traceEnd();

            // Make sure user is in the started state.  If it is currently
            // stopping, we need to knock that off.
            if (uss.state == UserState.STATE_STOPPING) {
                t.traceBegin("updateStateStopping");
                // If we are stopping, we haven't sent ACTION_SHUTDOWN,
                // so we can just fairly silently bring the user back from
                // the almost-dead.
                uss.setState(uss.lastState);
                mInjector.getUserManagerInternal().setUserState(userId, uss.state);
                synchronized (mLock) {
                    updateStartedUserArrayLU();
                }
                needStart = true;
                t.traceEnd();
            } else if (uss.state == UserState.STATE_SHUTDOWN) {
                t.traceBegin("updateStateShutdown");
                // This means ACTION_SHUTDOWN has been sent, so we will
                // need to treat this as a new boot of the user.
                uss.setState(UserState.STATE_BOOTING);
                mInjector.getUserManagerInternal().setUserState(userId, uss.state);
                synchronized (mLock) {
                    updateStartedUserArrayLU();
                }
                needStart = true;
                t.traceEnd();
            }

            if (uss.state == UserState.STATE_BOOTING) {
                t.traceBegin("updateStateBooting");
                // Give user manager a chance to propagate user restrictions
                // to other services and prepare app storage
                mInjector.getUserManager().onBeforeStartUser(userId);

                // Booting up a new user, need to tell system services about it.
                // Note that this is on the same handler as scheduling of broadcasts,
                // which is important because it needs to go first.
                mHandler.sendMessage(mHandler.obtainMessage(USER_START_MSG, userId, 0));
                t.traceEnd();
            }

            t.traceBegin("sendMessages");
            if (foreground) {
                mHandler.sendMessage(mHandler.obtainMessage(USER_CURRENT_MSG, userId, oldUserId));
                mHandler.removeMessages(REPORT_USER_SWITCH_MSG);
                mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
                mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_MSG,
                        oldUserId, userId, uss));
                mHandler.sendMessageDelayed(mHandler.obtainMessage(USER_SWITCH_TIMEOUT_MSG,
                        oldUserId, userId, uss), USER_SWITCH_TIMEOUT_MS);
            }

            if (userInfo.preCreated) {
                needStart = false;
            }

            if (needStart) {
                // Send USER_STARTED broadcast
                Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                mInjector.broadcastIntent(intent,
                        null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                        null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid, userId);
            }
            t.traceEnd();

            if (foreground) {
                t.traceBegin("moveUserToForeground");
                moveUserToForeground(uss, oldUserId, userId);
                t.traceEnd();
            } else {
                t.traceBegin("finishUserBoot");
                finishUserBoot(uss);
                t.traceEnd();
            }

            if (needStart) {
                t.traceBegin("sendRestartBroadcast");
                Intent intent = new Intent(Intent.ACTION_USER_STARTING);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                mInjector.broadcastIntent(intent,
                        null, new IIntentReceiver.Stub() {
                            @Override
                            public void performReceive(Intent intent, int resultCode,
                                    String data, Bundle extras, boolean ordered,
                                    boolean sticky,
                                    int sendingUser) throws RemoteException {
                            }
                        }, 0, null, null,
                        new String[]{INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                        UserHandle.USER_ALL);
                t.traceEnd();
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }

        return true;
    }
```





