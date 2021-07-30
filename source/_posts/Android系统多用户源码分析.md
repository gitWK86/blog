# Android系统多用户源码分析

## 概述

Android从4.2开始支持多用户模式，不同的用户运行在独立的用户空间，它们的的锁屏设置，PIN码和壁纸等系统设置是各不相同的，而且不同用户安装的应用和应用数据都是不一样的，但是系统中和硬件相关的设置则是共用的，比如网络设置等。通常第一个在系统中注册的用户将成为系统管理员，可以管理手机上的其他用户。

手机上经常见到的一个功能就是访客模式，它是多用户模式的一种应用。访客模式下用户的所有数据（通讯录，短信，应用等）会被隐藏，访客只能查看和使用手机的基本功能，另外你还可以设置访客是否有接听电话、发送短信等权限。

## adb相关命令

1. 获取用户列表

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

## 架构图

![](Android系统多用户\multi-user.png)



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

## UserController

后面在用户切换、启动的时候我们会看到主要工作都交给了`UserController`，所以在这里先了解下`UserController`是做什么的。

`UserController`是AMS的Helper类，负责多用户启动、停止功能。

它的创建是在AMS的构造函数中

```java
mUserController = new UserController(this);
```

`UserController`构造函数

```java
UserController(ActivityManagerService service) {
        this(new Injector(service));
 }
 
 UserController(Injector injector) {
        mInjector = injector;
        mHandler = mInjector.getHandler(this);
        mUiHandler = mInjector.getUiHandler(this);
        // User 0 is the first and only user that runs at boot.
        final UserState uss = new UserState(UserHandle.SYSTEM);
        uss.mUnlockProgress.addListener(new UserProgressListener());
        // 已启动用户列表
        mStartedUsers.put(UserHandle.USER_SYSTEM, uss);
        mUserLru.add(UserHandle.USER_SYSTEM);
        mLockPatternUtils = mInjector.getLockPatternUtils();
        updateStartedUserArrayLU();
  }
```

初始化的时候创建了一个`UserController#Injector`对象，传入了AMS实例，后面在很多地方会调用AMS的方法，都通过这个对象实现。

### Injector

```java
static class Injector {
        private final ActivityManagerService mService;
        private UserManagerService mUserManager;
        private UserManagerInternal mUserManagerInternal;

        Injector(ActivityManagerService service) {
            mService = service;
        }
        ......
        protected int broadcastIntent(){
            ......
        }
        
        SystemServiceManager getSystemServiceManager() {
            return mService.mSystemServiceManager;
        }
        ......
 }
```



## UserState

先了解一下用户的几个运行状态：

```java
public final class UserState {

    // 用户第一次启动
    public final static int STATE_BOOTING = 0;
    // 用户处于锁定状态
    public final static int STATE_RUNNING_LOCKED = 1;
    // 用户处于解锁状态
    public final static int STATE_RUNNING_UNLOCKING = 2;
    // 用户处于正在运行状态
    public final static int STATE_RUNNING_UNLOCKED = 3;
    // 用户处于开始被停止状态
    public final static int STATE_STOPPING = 4;
    // 用户处于被停止状态, 发送 Intent.ACTION_SHUTDOWN广播
    public final static int STATE_SHUTDOWN = 5;
    
}
```



## 创建新用户

### 创建时序图



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
                    // 正在创建标记
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

`am switch-user user-id`切换用户的命令实际执行的也是`start-user`的流程，它只是会有一个提示用户切换的弹窗，在`start-user`中执行切换前台的逻辑。

这里主要看下`start-user`的流程



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

            // 如果是前台启动，则需要将屏幕冻结
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
                // 创建该用户的UserState对象，用于后续启动期间，各个状态的保存和切换 
                // 第一次执行的话，该对象为null，新创建的对象的状态处于初始状态0(BOOTING)
                uss = mStartedUsers.get(userId);
                if (uss == null) {
                    uss = new UserState(UserHandle.of(userId));
                    uss.mUnlockProgress.addListener(new UserProgressListener());
                    // 记录启动过的用户
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
                // 如果是前台切换
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
                // 在WindowManagerService中设置要启动的用户为当前用户
                mInjector.getWindowManager().setCurrentUser(userId, getCurrentProfileIds());
                mInjector.reportCurWakefulnessUsageEvent();
                // Once the internal notion of the active user has switched, we lock the device
                // with the option to show the user switcher on the keyguard.
                if (userSwitchUiEnabled) {
                    // 提示用户切换并锁屏
                    mInjector.getWindowManager().setSwitchingUser(true);
                    mInjector.getWindowManager().lockNow(null);
                }
            } else {
                // 如果是后台启动
                final Integer currentUserIdInt = mCurrentUserId;
                // 当发生用户切换或在后台启动新的相关用户时，刷新与当前用户相关的用户列表。
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
                // 如果该用户是正在停止，这个时候还没有发送ACTION_SHUTDOWN广播，则切换为正在运行
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
                // 如果该用户是已经停止，已经发送了ACTION_SHUTDOWN广播，则切换为正在启动状态
                uss.setState(UserState.STATE_BOOTING);
                mInjector.getUserManagerInternal().setUserState(userId, uss.state);
                synchronized (mLock) {
                    updateStartedUserArrayLU();
                }
                needStart = true;
                t.traceEnd();
            }

            // 发送SYSTEM_USER_START_MSG消息，回调系统指所有systemServer的onStartUser方法，通知user启动
            if (uss.state == UserState.STATE_BOOTING) {
                t.traceBegin("updateStateBooting");
                // Give user manager a chance to propagate user restrictions
                // to other services and prepare app storage
                mInjector.getUserManager().onBeforeStartUser(userId);

                // Booting up a new user, need to tell system services about it.
                // Note that this is on the same handler as scheduling of broadcasts,
                // which is important because it needs to go first.
                // 通知BatteryStatsService用户切换的消息以及让SystemServiceManager通知各个
                // SystemService调用onUserStarting
                mHandler.sendMessage(mHandler.obtainMessage(USER_START_MSG, userId, 0));
                t.traceEnd();
            }

            t.traceBegin("sendMessages");
            if (foreground) {
                // 通知BatteryStatsService用户切换信息以及让SystemServiceManager通知各个
                // SeysteService调用switchUser
                mHandler.sendMessage(mHandler.obtainMessage(USER_CURRENT_MSG, userId, oldUserId));
                mHandler.removeMessages(REPORT_USER_SWITCH_MSG);
                mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
                // 如果系统中有对切换用户感兴趣的模块，可以调用AMS的registerUserSwitchObserver方法
                // 来注册观察对象
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
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY //静态注册的广播不能接受该广播
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                mInjector.broadcastIntent(intent,
                        null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                        null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid, userId);
            }
            t.traceEnd();

            if (foreground) {
                t.traceBegin("moveUserToForeground");
                // 设置为前台用户
                moveUserToForeground(uss, oldUserId, userId);
                t.traceEnd();
            } else {
                t.traceBegin("finishUserBoot");
                // 执行Boot操作，会执行执行user启动的各个状态，并报个每个阶段的进度
                finishUserBoot(uss);
                t.traceEnd();
            }

            if (needStart) {
                // 发送ACTION_USER_STARTING广播
                t.traceBegin("sendRestartBroadcast");
                Intent intent = new Intent(Intent.ACTION_USER_STARTING);
                //静态注册的广播不能接受该广播
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

### 启动时序图

启动流程总结如下

![](Android系统多用户\amStartUesr.png)



1. 创建改用户对象的UserState，第一次启动时，该用户的状态为0(STATE_BOOTING)
2. 保存主用户和该用户对应的profileGroupid的对应关系，维护一个{userid,profileGroupId}的数据结构
3. 发送USER_START_MSG消息，通知BatteryStatsService用户切换的消息以及让SystemServiceManager通知各个SystemService调用onUserStarting
4. Send ACTION_USER_STARTED broadcast
5. 执行finishUserBoot操作，结束Boot状态
6. 发送ACTION_USER_STARTING广播

### 消息处理

#### USER_START_MSG

通知BatteryStatsService用户切换的消息以及让SystemServiceManager通知各个SystemService调用onUserStarting

```java
case USER_START_MSG:
        mInjector.batteryStatsServiceNoteEvent(
        BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
        Integer.toString(msg.arg1), msg.arg1);

        mInjector.getSystemServiceManager().startUser(TimingsTraceAndSlog.newAsyncLog(),
        msg.arg1);

        break;
```

`frameworks/base/services/core/java/com/android/server/SystemServiceManager.java`

```java
public void startUser(final @NonNull TimingsTraceAndSlog t, final @UserIdInt int userHandle) {
        onUser(t, START, userHandle);
}

private void onUser(@NonNull TimingsTraceAndSlog t, @NonNull String onWhat,
            @UserIdInt int curUserId, @UserIdInt int prevUserId) {
        final TargetUser curUser = new TargetUser(getUserInfo(curUserId));
        final TargetUser prevUser = prevUserId == UserHandle.USER_NULL ? null
                : new TargetUser(getUserInfo(prevUserId));
        final int serviceLen = mServices.size();
        for (int i = 0; i < serviceLen; i++) {
            ......
            final SystemService service = mServices.get(i);
            try {
                switch (onWhat) {
                    case SWITCH:
                        service.onUserSwitching(prevUser, curUser);
                        break;
                    case START:
                        service.onUserStarting(curUser);
                        break;
                    case UNLOCKING:
                        service.onUserUnlocking(curUser);
                        break;
                    case UNLOCKED:
                        service.onUserUnlocked(curUser);
                        break;
                    case STOP:
                        service.onUserStopping(curUser);
                        break;
                    case CLEANUP:
                        service.onUserStopped(curUser);
                        break;
                    default:
                        throw new IllegalArgumentException(onWhat + " what?");
                }
            } catch (Exception ex) {
                Slog.wtf(TAG, "Failure reporting " + onWhat + " of user " + curUser
                        + " to service " + serviceName, ex);
            }
        }
    }
```

当我们要切换用户到前台时，下面几个消息处理就会被执行

#### USER_CURRENT_MSG

SystemServiceManager通知各个SystemService调用onUserSwitching

```java
case USER_CURRENT_MSG:
        mInjector.batteryStatsServiceNoteEvent(
        BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_FINISH,
        Integer.toString(msg.arg2), msg.arg2);

        mInjector.batteryStatsServiceNoteEvent(
        BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
        Integer.toString(msg.arg1), msg.arg1);

        mInjector.getSystemServiceManager().switchUser(msg.arg2, msg.arg1);
        break;
```



#### REPORT_USER_SWITCH_MSG

如果系统中有对切换用户感兴趣的模块（例如：`BiometricService`、`PowerManagerService`、`WallpaperManagerService`），可以调用AMS的`registerUserSwitchObserver`方法来注册观察对象，对象会保存在`mUserSwitchObservers`中。模块中处理完成后会调用参数中传递的`callback`来通知AMS。最后所有模块调用完成后会调用`sendContinueUserSwitchLU`来继续进行切换用户的工作。

注册流程：

`frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`

```java
@Override
public void registerUserSwitchObserver(IUserSwitchObserver observer, String name) {
	mUserController.registerUserSwitchObserver(observer, name);
}
```

`frameworks/base/services/core/java/com/android/server/am/UserController.java`

```java
void registerUserSwitchObserver(IUserSwitchObserver observer, String name) {
        Objects.requireNonNull(name, "Observer name cannot be null");
        checkCallingPermission(INTERACT_ACROSS_USERS_FULL, "registerUserSwitchObserver");
        mUserSwitchObservers.register(observer, name);
    }
```

处理流程：

`frameworks/base/services/core/java/com/android/server/am/UserController.java`

```java
case REPORT_USER_SWITCH_MSG:
        dispatchUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
        break;
```

```java
void dispatchUserSwitch(final UserState uss, final int oldUserId, final int newUserId) {

        final int observerCount = mUserSwitchObservers.beginBroadcast();
        if (observerCount > 0) {
            final ArraySet<String> curWaitingUserSwitchCallbacks = new ArraySet<>();
            synchronized (mLock) {
                uss.switching = true;
                mCurWaitingUserSwitchCallbacks = curWaitingUserSwitchCallbacks;
            }
            final AtomicInteger waitingCallbacksCount = new AtomicInteger(observerCount);
            final long dispatchStartedTime = SystemClock.elapsedRealtime();
            for (int i = 0; i < observerCount; i++) {
                try {
                    // 添加唯一前缀以保证键是唯一的
                    final String name = "#" + i + " " + mUserSwitchObservers.getBroadcastCookie(i);
                    synchronized (mLock) {
                        curWaitingUserSwitchCallbacks.add(name);
                    }
                    final IRemoteCallback callback = new IRemoteCallback.Stub() {
                        @Override
                        public void sendResult(Bundle data) throws RemoteException {
                            synchronized (mLock) {
                                long delay = SystemClock.elapsedRealtime() - dispatchStartedTime;
                                
                                curWaitingUserSwitchCallbacks.remove(name);
                                // Continue switching if all callbacks have been notified and
                                // user switching session is still valid
                                if (waitingCallbacksCount.decrementAndGet() == 0
                                        && (curWaitingUserSwitchCallbacks
                                        == mCurWaitingUserSwitchCallbacks)) {
                                    //待所有结果都返回了，发送继续处理的消息
                                    sendContinueUserSwitchLU(uss, oldUserId, newUserId);
                                }
                            }
                        }
                    };
                    // 向注册的对象通知onUserSwitching，
                    // 当注册的模块处理完用户切换到流程后再调用callback返回数据。
                    mUserSwitchObservers.getBroadcastItem(i).onUserSwitching(newUserId, callback);
                } catch (RemoteException e) {
                }
            }
        } else {
            synchronized (mLock) {
                sendContinueUserSwitchLU(uss, oldUserId, newUserId);
            }
        }
        mUserSwitchObservers.finishBroadcast();
    }

void sendContinueUserSwitchLU(UserState uss, int oldUserId, int newUserId) {
        mCurWaitingUserSwitchCallbacks = null;
        // 移除超时信息
        mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
        mHandler.sendMessage(mHandler.obtainMessage(CONTINUE_USER_SWITCH_MSG,
                oldUserId, newUserId, uss));
    }
```

#### CONTINUE_USER_SWITCH_MSG

```java
case CONTINUE_USER_SWITCH_MSG:
     continueUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
     break;


void continueUserSwitch(UserState uss, int oldUserId, int newUserId) {
        if (isUserSwitchUiEnabled()) {
            // 结束冻屏
            mInjector.getWindowManager().stopFreezingScreen();
        }
        // 标记用户切换完成
        uss.switching = false;
        // 发送切换完成的消息
        mHandler.removeMessages(REPORT_USER_SWITCH_COMPLETE_MSG);
        mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_COMPLETE_MSG, newUserId, 0));
        // 停止后台用户
        stopGuestOrEphemeralUserIfBackground(oldUserId);
        stopBackgroundUsersIfEnforced(oldUserId);
 }
```

#### REPORT_USER_SWITCH_COMPLETE_MSG

这个消息主要工作就是调用观察者的`onUserSwitchComplete()`方法，进行用户切换的收尾工作。

```java
/** Called on handler thread */
    void dispatchUserSwitchComplete(@UserIdInt int userId) {
        mInjector.getWindowManager().setSwitchingUser(false);
        final int observerCount = mUserSwitchObservers.beginBroadcast();
        for (int i = 0; i < observerCount; i++) {
            try {
                mUserSwitchObservers.getBroadcastItem(i).onUserSwitchComplete(userId);
            } catch (RemoteException e) {
            }
        }
        mUserSwitchObservers.finishBroadcast();
    }
```

### finishUserBoot()

在上面的流程中，我们的用户状态还是在`UserState.STATE_BOOTING`，那么如何将User进入下个阶段呢？

```java
if (foreground) {
    // 设置为前台用户
    moveUserToForeground(uss, oldUserId, userId);
} else {
   // 执行Boot操作，会执行执行user启动的各个状态，并报个每个阶段的进度
   finishUserBoot(uss);
}
```

先看下`finishUserBoot`的逻辑，`finishUserBoot`将新创建的User从状态`STATE_BOOTING`设置为`STATE_RUNNING_LOCKED`，最后调用`maybeUnlockUser()`

```java
    private void finishUserBoot(UserState uss) {
        finishUserBoot(uss, null);
    }

    private void finishUserBoot(UserState uss, IIntentReceiver resultTo) {
        final int userId = uss.mHandle.getIdentifier();
        // We always walk through all the user lifecycle states to send
        // consistent developer events. We step into RUNNING_LOCKED here,
        // but we might immediately step into RUNNING below if the user
        // storage is already unlocked.
        // 遍历所有用户生命周期状态，以发送一致的事件。 我们在这里进入RUNNING_LOCKED，
        // 但如果用户存储已经解锁，我们可能会立即进入下面的RUNNING状态。  
        if (uss.setState(STATE_BOOTING, STATE_RUNNING_LOCKED)) {
            logUserLifecycleEvent(userId, USER_LIFECYCLE_EVENT_USER_RUNNING_LOCKED,
                    USER_LIFECYCLE_EVENT_STATE_NONE);
            mInjector.getUserManagerInternal().setUserState(userId, uss.state);

            // 不是预创建的用户
            // 发送REPORT_LOCKED_BOOT_COMPLETE_MSG消息，向注册的对象通知onLockedBootComplete
            // 发送ACTION_LOCKED_BOOT_COMPLETED广播
            if (!mInjector.getUserManager().isPreCreated(userId)) {
                mHandler.sendMessage(mHandler.obtainMessage(REPORT_LOCKED_BOOT_COMPLETE_MSG,
                        userId, 0));
                Intent intent = new Intent(Intent.ACTION_LOCKED_BOOT_COMPLETED, null);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                intent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
                        | Intent.FLAG_RECEIVER_OFFLOAD
                        | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
                mInjector.broadcastIntent(intent, null, resultTo, 0, null, null,
                        new String[]{android.Manifest.permission.RECEIVE_BOOT_COMPLETED},
                        AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID,
                        Binder.getCallingUid(), Binder.getCallingPid(), userId);
            }
        }

        // We need to delay unlocking managed profiles until the parent user
        // is also unlocked.
        // 检查调用上下文用户是否在配置文件中运行
        if (mInjector.getUserManager().isProfile(userId)) {
            final UserInfo parent = mInjector.getUserManager().getProfileParent(userId);
            if (parent != null
                    && isUserRunning(parent.id, ActivityManager.FLAG_AND_UNLOCKED)) {
                Slog.d(TAG, "User " + userId + " (parent " + parent.id
                        + "): attempting unlock because parent is unlocked");
                maybeUnlockUser(userId);
            } else {
                String parentId = (parent == null) ? "<null>" : String.valueOf(parent.id);
                Slog.d(TAG, "User " + userId + " (parent " + parentId
                        + "): delaying unlock because parent is locked");
            }
        } else {
            maybeUnlockUser(userId);
        }
    }
```

#### maybeUnlockUser()

```java
private boolean maybeUnlockUser(final @UserIdInt int userId) {
    // Try unlocking storage using empty token
    return unlockUserCleared(userId, null, null, null);
}
```

#### unlockUserCleared()

User状态从`STATE_RUNNING_LOCKED`设置为`STATE_RUNNING_UNLOCKING`，发送`USER_UNLOCK_MSG`消息进行下一步操作。

```java
    /**
     * Step from {@link UserState#STATE_RUNNING_LOCKED} to
     * {@link UserState#STATE_RUNNING_UNLOCKING}.
     */
    private boolean finishUserUnlocking(final UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        // 解锁进度
        uss.mUnlockProgress.start();

        // Prepare app storage before we go any further
        // 进度5%
        uss.mUnlockProgress.setProgress(5,
                    mInjector.getContext().getString(R.string.android_start_title));

        // Call onBeforeUnlockUser on a worker thread that allows disk I/O
        FgThread.getHandler().post(() -> {
            mInjector.getUserManager().onBeforeUnlockUser(userId);
            synchronized (mLock) {
                // Do not proceed if unexpected state
                if (!uss.setState(STATE_RUNNING_LOCKED, STATE_RUNNING_UNLOCKING)) {
                    return;
                }
            }
            mInjector.getUserManagerInternal().setUserState(userId, uss.state);

            // 进度20%
            uss.mUnlockProgress.setProgress(20);

            // Dispatch unlocked to system services; when fully dispatched,
            // that calls through to the next "unlocked" phase
            // 发送USER_UNLOCK_MSG消息
            mHandler.obtainMessage(USER_UNLOCK_MSG, userId, 0, uss).sendToTarget();
        });
        return true;
    }
```

#### USER_UNLOCK_MSG

```java
       case USER_UNLOCK_MSG:
                final int userId = msg.arg1;
                // 回调系统所有systemServer的onUserUnlocking方法
                mInjector.getSystemServiceManager().unlockUser(userId);
                // Loads recents on a worker thread that allows disk I/O
                FgThread.getHandler().post(() -> {
                    mInjector.loadUserRecents(userId);
                });
                
                finishUserUnlocked((UserState) msg.obj);
                break;
```

#### finishUserUnlocked()

User状态从`STATE_RUNNING_UNLOCKING`设置为`STATE_RUNNING_UNLOCKED`，最后将调用`finishUserUnlockedCompleted`方法

```java
     /**
     * Step from {@link UserState#STATE_RUNNING_UNLOCKING} to
     * {@link UserState#STATE_RUNNING_UNLOCKED}.
     */
    void finishUserUnlocked(final UserState uss) {
        final int userId = uss.mHandle.getIdentifier();

        synchronized (mLock) {

            // Do not proceed if unexpected state
            if (!uss.setState(STATE_RUNNING_UNLOCKING, STATE_RUNNING_UNLOCKED)) {
                return;
            }
        }
        mInjector.getUserManagerInternal().setUserState(userId, uss.state);
        // UnlockProgress完成
        uss.mUnlockProgress.finish();

        // Dispatch unlocked to external apps
        final Intent unlockedIntent = new Intent(Intent.ACTION_USER_UNLOCKED);
        unlockedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
        unlockedIntent.addFlags(
                Intent.FLAG_RECEIVER_REGISTERED_ONLY | Intent.FLAG_RECEIVER_FOREGROUND);
        mInjector.broadcastIntent(unlockedIntent, null, null, 0, null,
                null, null, AppOpsManager.OP_NONE, null, false, false, MY_PID, SYSTEM_UID,
                Binder.getCallingUid(), Binder.getCallingPid(), userId);

        // Send PRE_BOOT broadcasts if user fingerprint changed; we
        // purposefully block sending BOOT_COMPLETED until after all
        // PRE_BOOT receivers are finished to avoid ANR'ing apps
        final UserInfo info = getUserInfo(userId);
        if (!Objects.equals(info.lastLoggedInFingerprint, Build.FINGERPRINT)) {
            // Suppress double notifications for managed profiles that
            // were unlocked automatically as part of their parent user
            // being unlocked.
            final boolean quiet;
            if (info.isManagedProfile()) {
                quiet = !uss.tokenProvided
                        || !mLockPatternUtils.isSeparateProfileChallengeEnabled(userId);
            } else {
                quiet = false;
            }
            mInjector.sendPreBootBroadcast(userId, quiet,
                    () -> finishUserUnlockedCompleted(uss));
        } else {
            finishUserUnlockedCompleted(uss);
        }
    }
```

#### finishUserUnlockedCompleted()

在`finishUserUnlockedCompleted`中如果新建的User没有被`initialized`。会调用`getUserManager().makeInitialized`方法完成新建User的initialized。同时发出`ACTION_BOOT_COMPLETED`。至此就完成了新建User的启动。

```java
    private void finishUserUnlockedCompleted(UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
       
        UserInfo userInfo = getUserInfo(userId);
        if (userInfo == null) {
            return;
        }
        // Only keep marching forward if user is actually unlocked
        if (!StorageManager.isUserKeyUnlocked(userId)) return;

        // Remember that we logged in
        mInjector.getUserManager().onUserLoggedIn(userId);

        if (!userInfo.isInitialized()) {
            if (userId != UserHandle.USER_SYSTEM) {
                Slog.d(TAG, "Initializing user #" + userId);
                Intent intent = new Intent(Intent.ACTION_USER_INITIALIZE);
                intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND
                        | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
                mInjector.broadcastIntent(intent, null,
                        new IIntentReceiver.Stub() {
                            @Override
                            public void performReceive(Intent intent, int resultCode,
                                    String data, Bundle extras, boolean ordered,
                                    boolean sticky, int sendingUser) {
                                // Note: performReceive is called with mService lock held
                                mInjector.getUserManager().makeInitialized(userInfo.id);
                            }
                        }, 0, null, null, null, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
                        Binder.getCallingPid(), userId);
            }
        }


        // Spin up app widgets prior to boot-complete, so they can be ready promptly
        mInjector.startUserWidgets(userId);

        // 回调系统所有systemService的onUserUnlocked方法
        mHandler.obtainMessage(USER_UNLOCKED_MSG, userId, 0).sendToTarget();

        Slog.i(TAG, "Posting BOOT_COMPLETED user #" + userId);
        
        // 发出ACTION_BOOT_COMPLETED广播
        final Intent bootIntent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
        bootIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
        bootIntent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
                | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND
                | Intent.FLAG_RECEIVER_OFFLOAD);
        // Widget broadcasts are outbound via FgThread, so to guarantee sequencing
        // we also send the boot_completed broadcast from that thread.
        final int callingUid = Binder.getCallingUid();
        final int callingPid = Binder.getCallingPid();
        FgThread.getHandler().post(() -> {
            mInjector.broadcastIntent(bootIntent, null,
                    new IIntentReceiver.Stub() {
                        @Override
                        public void performReceive(Intent intent, int resultCode, String data,
                                Bundle extras, boolean ordered, boolean sticky, int sendingUser)
                                        throws RemoteException {
                            Slog.i(UserController.TAG, "Finished processing BOOT_COMPLETED for u"
                                    + userId);
                            mBootCompleted = true;
                        }
                    }, 0, null, null,
                    new String[]{android.Manifest.permission.RECEIVE_BOOT_COMPLETED},
                    AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID,
                    callingUid, callingPid, userId);
        });
    }

```

然后再看下`moveUserToForeground()`

### moveUserToForeground()

```java
    private void moveUserToForeground(UserState uss, int oldUserId, int newUserId) {
        boolean homeInFront = mInjector.stackSupervisorSwitchUser(newUserId, uss);
        if (homeInFront) {
            mInjector.startHomeActivity(newUserId, "moveUserToForeground");
        } else {
            mInjector.stackSupervisorResumeFocusedStackTopActivity();
        }
        EventLogTags.writeAmSwitchUser(newUserId);
        sendUserSwitchBroadcasts(oldUserId, newUserId);
    }
```

除了`sendUserSwitchBroadcasts`就是发送广播通知新旧用户进行了切换。

剩下的操作就是去启动Activity了，为什么要去启动一个Activity呢？

我们知道Activity的启动流程中，在`onResume`后会调用`ActivityTaskManagerService`的`activityIdle`方法，然后调用`ActivityStackSupervisor`的`activityIdleInternal`方法去进行一些内存回收操作。

`frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java`

```java
void activityIdleInternal(ActivityRecord r, boolean fromTimeout,
            boolean processPausingActivities, Configuration config) {
        boolean booting = false;
    
        ......
            
        if (!mStartingUsers.isEmpty()) {
            final ArrayList<UserState> startingUsers = new ArrayList<>(mStartingUsers);
            mStartingUsers.clear();

            if (!booting) {
                // Complete user switch.
                for (int i = 0; i < startingUsers.size(); i++) {
                    mService.mAmInternal.finishUserSwitch(startingUsers.get(i));
                }
            }
        }

        mService.mH.post(() -> mService.mAmInternal.trimApplications());
    }
```

最后又会调用到`UserController`

`frameworks/base/services/core/java/com/android/server/am/UserController.java`

```java
  void finishUserSwitch(UserState uss) {
        // This call holds the AM lock so we post to the handler.
        mHandler.post(() -> {
            finishUserBoot(uss);
            startProfiles();
            synchronized (mLock) {
                stopRunningUsersLU(mMaxRunningUsers);
            }
        });
    }
```

这样又调用回了上面`finishUserBoot`的逻辑去完成用户的状态更改。



## 删除用户

### 删除用户时序图

![](Android系统多用户\pmRemoveUser.png)

删除用户是在UMS的`removeUserUnchecked()`方法中实现的

```java
    private boolean removeUserUnchecked(@UserIdInt int userId) {
        long ident = Binder.clearCallingIdentity();
        try {
            final UserData userData;
            int currentUser = ActivityManager.getCurrentUser();
            // 当前用户不能被删除
            if (currentUser == userId) {
                Slog.w(LOG_TAG, "Current user cannot be removed.");
                return false;
            }
            synchronized (mPackagesLock) {
                synchronized (mUsersLock) {
                    userData = mUsers.get(userId);
                    // 系统用户不能被删除
                    if (userId == UserHandle.USER_SYSTEM) {
                        Slog.e(LOG_TAG, "System user cannot be removed.");
                        return false;
                    }

                    // 用户不存在
                    if (userData == null) {
                        Slog.e(LOG_TAG, String.format(
                                "Cannot remove user %d, invalid user id provided.", userId));
                        return false;
                    }

                    // 用户正在删除中
                    if (mRemovingUserIds.get(userId)) {
                        Slog.e(LOG_TAG, String.format(
                                "User %d is already scheduled for removal.", userId));
                        return false;
                    }

                    // 将该用户放入mRemovingUserIds列表中，防止重复删除
                    // mRemovingUserIds中的数据会一直保存直到系统重启，防止Id被重复使用
                    addRemovingUserIdLocked(userId);
                }

                // 将partial设置为true，如果后面的过程意外终止导致此次删除失败,
                // 系统重启后还是会继续删除工作的
                userData.info.partial = true;
                // 设置FLAG_DISABLED，禁止该用户
                userData.info.flags |= UserInfo.FLAG_DISABLED;
                // 将上面更新的用户文件信息写入到xml文件中去
                writeUserLP(userData);
            }
            try {
                mAppOpsService.removeUser(userId);
            } catch (RemoteException e) {
                Slog.w(LOG_TAG, "Unable to notify AppOpsService of removing user.", e);
            }

            // 如果该user是一个user的一份profile，则发出一个ACTION_MANAGED_PROFILE_REMOVED广播
            if (userData.info.profileGroupId != UserInfo.NO_PROFILE_GROUP_ID
                    && userData.info.isManagedProfile()) {
                // Send broadcast to notify system that the user removed was a
                // managed user.
                sendProfileRemovedBroadcast(userData.info.profileGroupId, userData.info.id);
            }

            if (DBG) Slog.i(LOG_TAG, "Stopping user " + userId);
            int res;
            try {
                // 调用AMS停止当前的用户
                res = ActivityManager.getService().stopUser(userId, /* force= */ true,
                new IStopUserCallback.Stub() {
                            @Override
                            public void userStopped(int userIdParam) {
                                // 设置回调函数，调用finishRemoveUser继续后面的删除工作
                                finishRemoveUser(userIdParam);
                            }
                            @Override
                            public void userStopAborted(int userIdParam) {
                            }
                        });
            } catch (RemoteException e) {
                Slog.w(LOG_TAG, "Failed to stop user during removal.", e);
                return false;
            }
            return res == ActivityManager.USER_OP_SUCCESS;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

我们先看UMS中的代码，AMS稍后梳理

### finishRemoveUser()

```java
   void finishRemoveUser(final @UserIdInt int userId) {
        if (DBG) Slog.i(LOG_TAG, "finishRemoveUser " + userId);
        // 在完全清除用户的系统目录并从用户列表中删除之前，让其他服务关闭任何活动并清理它们的状态
        long ident = Binder.clearCallingIdentity();
        try {
            Intent removedIntent = new Intent(Intent.ACTION_USER_REMOVED);
            removedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
            // Also, add the UserHandle for mainline modules which can't use the @hide
            // EXTRA_USER_HANDLE.
            removedIntent.putExtra(Intent.EXTRA_USER, UserHandle.of(userId));
            mContext.sendOrderedBroadcastAsUser(removedIntent, UserHandle.ALL,
                    android.Manifest.permission.MANAGE_USERS,
                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                           
                            new Thread() {
                                @Override
                                public void run() {
                                    LocalServices.getService(ActivityManagerInternal.class)
                                            .onUserRemoved(userId);
                                    removeUserState(userId);
                                }
                            }.start();
                        }
                    },

                    null, Activity.RESULT_OK, null, null);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

根据代码可以看到在`finishRemoveUser`方法只是发送了一个有序广播`ACTION_USER_REMOVED`，同时注册了一个广播接收器，这个广播接收器是最后一个接收到该广播的接收器，这样做的目的是让关心该广播的其他接收器处理完之后， UMS 才会进行删除用户的收尾工作，即调用`removeUserState`来删除用户的相关文件。

### removeUserState()

```java
 private void removeUserState(final @UserIdInt int userId) {
    
        // 通过vold销毁文件系统加密
        mContext.getSystemService(StorageManager.class).destroyUserKey(userId);
      
        // Cleanup gatekeeper secure user id
        try {
            final IGateKeeperService gk = GateKeeper.getService();
            if (gk != null) {
                gk.clearSecureUserId(userId);
            }
        } catch (Exception ex) {
            Slog.w(LOG_TAG, "unable to clear GK secure user id");
        }

        // Cleanup package manager settings
        // 删除"/data/user/${userId}/${packageName}"目录
        mPm.cleanUpUser(this, userId);

        //删除 "/data/system/users/${userId}"
        // "/data/misc/users/${userId}"
        mUserDataPreparer.destroyUserData(userId,
                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);

        // 从列表中移除
        synchronized (mUsersLock) {
            mUsers.remove(userId);
            mIsUserManaged.delete(userId);
        }
        synchronized (mUserStates) {
            mUserStates.delete(userId);
        }
        synchronized (mRestrictionsLock) {
            mBaseUserRestrictions.remove(userId);
            mAppliedUserRestrictions.remove(userId);
            mCachedEffectiveUserRestrictions.remove(userId);
            // Remove local restrictions affecting user
            mDevicePolicyLocalUserRestrictions.delete(userId);
            // Remove local restrictions set by user
            boolean changed = false;
            for (int i = 0; i < mDevicePolicyLocalUserRestrictions.size(); i++) {
                int targetUserId = mDevicePolicyLocalUserRestrictions.keyAt(i);
                changed |= getDevicePolicyLocalRestrictionsForTargetUserLR(targetUserId)
                        .remove(userId);
            }
            changed |= mDevicePolicyGlobalUserRestrictions.remove(userId);
            if (changed) {
                applyUserRestrictionsForAllUsersLR();
            }
        }
        // Update the user list
        synchronized (mPackagesLock) {
            // 更新userlist.xml
            writeUserListLP();
        }
        // Remove user file
        AtomicFile userFile = new AtomicFile(new File(mUsersDir, userId + XML_SUFFIX));
        userFile.delete();
        // 更新所有用户缓存
        updateUserIds();
        if (RELEASE_DELETED_USER_ID) {
            synchronized (mUsers) {
                // 从删除列表中移除
                mRemovingUserIds.delete(userId);
            }
        }
    }
```

可以看到`removeUserState`中将我们在创建用户时产生的文件全部删除，使用PMS将生成的所有`/data/user/${userId}/${packageName}`文件夹删除，更新`userlist.xml`文件，将用户的所有信息清除。基本就是`create-user`的反向操作。

接下来再来看中间AMS做的操作：

### AMS.stopUser()

`kernel/android/R/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`

```java
    @Override
    public int stopUser(final int userId, boolean force, final IStopUserCallback callback) {
        return mUserController.stopUser(userId, force, /* allowDelayedLocking= */ false,
                /* callback= */ callback, /* keyEvictedCallback= */ null);
    }
```

AMS中直接调用了`UserController`的`stopUser()`方法

### UserController.stopUser()

```java
    int stopUser(final int userId, final boolean force, boolean allowDelayedLocking,
            final IStopUserCallback stopUserCallback, KeyEvictedCallback keyEvictedCallback) {
        checkCallingPermission(INTERACT_ACROSS_USERS_FULL, "stopUser");
        
        synchronized (mLock) {
            // 
            return stopUsersLU(userId, force, allowDelayedLocking, stopUserCallback,
                    keyEvictedCallback);
        }
    }
```

检查权限后调用`stopUsersLU()`

### stopUsersLU()

```java
    private int stopUsersLU(final int userId, boolean force, boolean allowDelayedLocking,
            final IStopUserCallback stopUserCallback, KeyEvictedCallback keyEvictedCallback) {
        // 系统用户，即0用户，不能删除
        if (userId == UserHandle.USER_SYSTEM) {
            return USER_OP_ERROR_IS_SYSTEM;
        }
        // 当前用户，不能删除
        if (isCurrentUserLU(userId)) {
            return USER_OP_IS_CURRENT;
        }
        // 确定应该与指定的用户一起停止的用户列表
        int[] usersToStop = getUsersToStopLU(userId);
        // 如果相关用户是系统用户或当前用户，则不应停止相关用户
        for (int i = 0; i < usersToStop.length; i++) {
            int relatedUserId = usersToStop[i];
            if ((UserHandle.USER_SYSTEM == relatedUserId) || isCurrentUserLU(relatedUserId)) {
                if (DEBUG_MU) Slog.i(TAG, "stopUsersLocked cannot stop related user "
                        + relatedUserId);
                // 如果是强制停止，我们仍然需要停止被请求的用户。  
                if (force) {
                    Slog.i(TAG,
                            "Force stop user " + userId + ". Related users will not be stopped");
                    stopSingleUserLU(userId, allowDelayedLocking, stopUserCallback,
                            keyEvictedCallback);
                    return USER_OP_SUCCESS;
                }
                return USER_OP_ERROR_RELATED_USERS_CANNOT_STOP;
            }
        }
        if (DEBUG_MU) Slog.i(TAG, "stopUsersLocked usersToStop=" + Arrays.toString(usersToStop));
        for (int userIdToStop : usersToStop) {
            stopSingleUserLU(userIdToStop, allowDelayedLocking,
                    userIdToStop == userId ? stopUserCallback : null,
                    userIdToStop == userId ? keyEvictedCallback : null);
        }
        return USER_OP_SUCCESS;
    }
```

`stopUsersLU()`中对要停止用户做了检查，不能删除系统用户和当前用户，然后调用`stopSingleUserLU()`

### stopSingleUserLU()

该方法会将User的状态设置为`STATE_STOPPING`。同时会调用了`finishUserStopping()`

```java
  private void stopSingleUserLU(final int userId, boolean allowDelayedLocking,
            final IStopUserCallback stopUserCallback,
            KeyEvictedCallback keyEvictedCallback) {
        if (DEBUG_MU) Slog.i(TAG, "stopSingleUserLocked userId=" + userId);
        final UserState uss = mStartedUsers.get(userId);
        if (uss == null) {  // 用户没有启动
             ......
            // 即使用户已经停止，也要发送用户停止回调
            if (stopUserCallback != null) {
                mHandler.post(() -> {
                    try {
                        stopUserCallback.userStopped(userId);
                    } catch (RemoteException e) {
                    }
                });
            }
            return;
        }

        if (stopUserCallback != null) {
            uss.mStopCallbacks.add(stopUserCallback);
        }
        if (keyEvictedCallback != null) {
            uss.mKeyEvictedCallbacks.add(keyEvictedCallback);
        }

        if (uss.state != UserState.STATE_STOPPING
                && uss.state != UserState.STATE_SHUTDOWN) {
            // 将用户状态设置为STATE_STOPPING
            uss.setState(UserState.STATE_STOPPING);
            mInjector.getUserManagerInternal().setUserState(userId, uss.state);
            updateStartedUserArrayLU();

            final boolean allowDelayyLockingCopied = allowDelayedLocking;
            // Post to handler to obtain amLock
            mHandler.post(() -> {
                // We are going to broadcast ACTION_USER_STOPPING and then
                // once that is done send a final ACTION_SHUTDOWN and then
                // stop the user.
                // 发送ACTION_USER_STOPPING广播
                final Intent stoppingIntent = new Intent(Intent.ACTION_USER_STOPPING);
                stoppingIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
       
                stoppingIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                stoppingIntent.putExtra(Intent.EXTRA_SHUTDOWN_USERSPACE_ONLY, true);
                // This is the result receiver for the initial stopping broadcast.
                final IIntentReceiver stoppingReceiver = new IIntentReceiver.Stub() {
                    @Override
                    public void performReceive(Intent intent, int resultCode, String data,
                            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                        mHandler.post(() -> finishUserStopping(userId, uss,
                                allowDelayyLockingCopied));
                    }
                };

                // Clear broadcast queue for the user to avoid delivering stale broadcasts
                mInjector.clearBroadcastQueueForUser(userId);
                // Kick things off.
                mInjector.broadcastIntent(stoppingIntent,
                        null, stoppingReceiver, 0, null, null,
                        new String[]{INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
                        Binder.getCallingPid(), UserHandle.USER_ALL);
            });
        }
    }
```

### finishUserStopping()

该方法中将用户状态设置为`STATE_SHUTDOWN`，发送`ACTION_SHUTDOWN`广播，回调系统所有`SystemService`的`onUserStopping`方法，最后调用了`finishUserStopped()`

```java
    void finishUserStopping(final int userId, final UserState uss,
            final boolean allowDelayedLocking) {
        EventLog.writeEvent(EventLogTags.UC_FINISH_USER_STOPPING, userId);
        // On to the next.
        final Intent shutdownIntent = new Intent(Intent.ACTION_SHUTDOWN);
        
        // This is the result receiver for the final shutdown broadcast.
        final IIntentReceiver shutdownReceiver = new IIntentReceiver.Stub() {
            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        finishUserStopped(uss, allowDelayedLocking);
                    }
                });
            }
        };

        synchronized (mLock) {
            if (uss.state != UserState.STATE_STOPPING) {
                // Whoops, we are being started back up.  Abort, abort!
                return;
            }
            uss.setState(UserState.STATE_SHUTDOWN);
        }
        mInjector.getUserManagerInternal().setUserState(userId, uss.state);

        mInjector.batteryStatsServiceNoteEvent(
                BatteryStats.HistoryItem.EVENT_USER_RUNNING_FINISH,
                Integer.toString(userId), userId);
        // 回调系统所有SystemService的onUserStopping方法
        mInjector.getSystemServiceManager().stopUser(userId);

        mInjector.broadcastIntent(shutdownIntent,
                null, shutdownReceiver, 0, null, null, null,
                AppOpsManager.OP_NONE,
                null, true, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
                Binder.getCallingPid(), userId);
    }
```

### finishUserStopped()

在`finishUserStopped`中主要是将多用户中的app停止，回调UMS执行最后的删除操作，回调系统所有`systemService`的`onUserStopped`方法通知用户删除，最后发送`ACTION_USER_STOPPED`广播。

```java
    void finishUserStopped(UserState uss, boolean allowDelayedLocking) {
        final int userId = uss.mHandle.getIdentifier();
        EventLog.writeEvent(EventLogTags.UC_FINISH_USER_STOPPED, userId);
        final boolean stopped;
        boolean lockUser = true;
        final ArrayList<IStopUserCallback> stopCallbacks;
        final ArrayList<KeyEvictedCallback> keyEvictedCallbacks;
        int userIdToLock = userId;
        synchronized (mLock) {
            stopCallbacks = new ArrayList<>(uss.mStopCallbacks);
            keyEvictedCallbacks = new ArrayList<>(uss.mKeyEvictedCallbacks);
            if (mStartedUsers.get(userId) != uss || uss.state != UserState.STATE_SHUTDOWN) {
                stopped = false;
            } else {
                stopped = true;
                // User can no longer run.
                mStartedUsers.remove(userId);
                mUserLru.remove(Integer.valueOf(userId));
                updateStartedUserArrayLU();
                if (allowDelayedLocking && !keyEvictedCallbacks.isEmpty()) {
                    Slog.wtf(TAG,
                            "Delayed locking enabled while KeyEvictedCallbacks not empty, userId:"
                                    + userId + " callbacks:" + keyEvictedCallbacks);
                    allowDelayedLocking = false;
                }
                userIdToLock = updateUserToLockLU(userId, allowDelayedLocking);
                if (userIdToLock == UserHandle.USER_NULL) {
                    lockUser = false;
                }
            }
        }
        if (stopped) {
            mInjector.getUserManagerInternal().removeUserState(userId);
            mInjector.activityManagerOnUserStopped(userId);
            // Clean up all state and processes associated with the user.
            // Kill all the processes for the user.
            forceStopUser(userId, "finish user");
        }

        for (final IStopUserCallback callback : stopCallbacks) {
            try {
                // 回调UMS执行删除操作
                if (stopped) callback.userStopped(userId);
                else callback.userStopAborted(userId);
            } catch (RemoteException ignored) {
            }
        }

        if (stopped) {
            // 回调系统所有SystemService的onUserStopped方法
            mInjector.systemServiceManagerCleanupUser(userId);
            mInjector.stackSupervisorRemoveUser(userId);
            // Remove the user if it is ephemeral.
            UserInfo userInfo = getUserInfo(userId);
            if (userInfo.isEphemeral() && !userInfo.preCreated) {
                mInjector.getUserManager().removeUserEvenWhenDisallowed(userId);
            }

            if (!lockUser) {
                return;
            }
            dispatchUserLocking(userIdToLock, keyEvictedCallbacks);
        }
    }
```

```java
   private void forceStopUser(@UserIdInt int userId, String reason) {
        //调用AMS的forceStopPackageLocked方法强行停止用户的app
        mInjector.activityManagerForceStopPackage(userId, reason);
        // 发送ACTION_USER_STOPPED广播
        Intent intent = new Intent(Intent.ACTION_USER_STOPPED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                | Intent.FLAG_RECEIVER_FOREGROUND);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
        mInjector.broadcastIntent(intent,
                null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                null, false, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
                Binder.getCallingPid(), UserHandle.USER_ALL);
    }
```

删除逻辑结束！

