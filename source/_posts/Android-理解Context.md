---

title: Android 理解Context
date: 2020-09-03 10:07:17
categories: 
- Android系统
tags:
- Android
- Context
---

接触过Android的小伙伴, 一定不会对Context感到陌生, 有大量的场景使用都离不开Context, 下面列举部分常见场景:

- 启动Activity (startActivity)
- 启动服务 (startService)
- 发送广播 (sendBroadcast), 注册广播接收者 (registerReceiver)
- 获取ContentResolver (getContentResolver)
- 获取类加载器 (getClassLoader)
- 打开或创建数据库 (openOrCreateDatabase)
- 获取资源 (getResources)
- …

### Context是什么

Context一般会被翻译为“上下文”，在Android中应该被翻译为“场景”。

一个Context意味着一个场景，一个场景就是用户和操作系统交互的一种过程。比如打电话时，场景包括电话程序对应的界面，以及隐藏在界面后的数据。

从代码来看，Activity类确实是基于Context，Service类也是基于Context。Activity除了基于Context外，还实现了一些其他重要接口，从设计角度上来说，interface仅仅是某些功能，extends才是类的本质，所以Activity和Service的本质就是一个Context。

Android Context本身是一个抽象类. ContextImpl, Activity, Service, Application这些都是Context的直接或间接子类, 下面通过看看这些类的关系,如下:(图片来源：[Gityuan](http://gityuan.com/2017/04/09/android_context/))

![](Android-理解Context/context.jpg)



### 一个应用程序包含多少个Context

在程序开发中，经常会调用Context的一些方法，这些方法会返回一个全局对象，而不是某个Activity，那一个程序到底有多少个Context对象？比如，Context.getResource()返回该应用程序的Resource对象，无论从哪个Activity中获取，都会返回同一个Resource对象。

这里可以明确：

- 一个Activity就是一个场景(Context)，一个Service也是一个场景，所以，一个程序有多少个Activity和Service，就有多少个Context对象 。
- getResource()等方法确实返回的是同一个全局对象。



### 创建Context

要理解Context, 需要依次来看看四大组件的初始化过程.

#### Activity

`frameworks/base/core/java/android/app/ActivityThread.java`

```java
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            //step 1: 创建LoadedApk对象
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        //component初始化过程
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //step 2: 创建Activity对象
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            //step 3: 创建Application对象
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                //step 4: 创建ContextImpl对象
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                //step5: 将Application/ContextImpl都attach到Activity对象 [下文详解]
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    //step 6: 执行回调onCreate
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    //执行回调onStart
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            //执行回调onRestoreInstanceState
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    
```

startActivity的过程最终会在目标进程执行performLaunchActivity()方法, 该方法主要功能:

1. 创建对象LoadedApk;
2. 创建对象Activity;
3. 创建对象Application;
4. 创建对象ContextImpl;
5. Application/ContextImpl都attach到Activity对象;
6. 执行onCreate()等回调;



#### Service

```java
private void handleCreateService(CreateServiceData data) {
      
        unscheduleGcIdler();

        //step 1: 创建LoadedApk
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
             //step 2: 创建Service对象
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            //step 3: 创建ContextImpl对象
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
          
            //step 4: 创建Application对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //step 5: 将Application/ContextImpl都attach到Activity对象
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //step 6: 执行onCreate回调
            service.onCreate();
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```

整个过程:

1. 创建对象LoadedApk;
2. 创建对象Service;
3. 创建对象ContextImpl;
4. 创建对象Application;
5. Application/ContextImpl分别attach到Service对象;
6. 执行onCreate()回调;

#### Receiver

```java
private void handleReceiver(ReceiverData data) {
        unscheduleGcIdler();

        String component = data.intent.getComponent().getClassName();

       //step 1: 创建LoadedApk对象
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManagerNative.getDefault();

        BroadcastReceiver receiver;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            //step 2: 创建BroadcastReceiver对象
            receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
        } catch (Exception e) {
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            data.sendFinished(mgr);
            throw new RuntimeException(
                "Unable to instantiate receiver " + component
                + ": " + e.toString(), e);
        }

        try {
            //step 3: 创建Application对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //step 4: 创建ContextImpl对象
            ContextImpl context = (ContextImpl)app.getBaseContext();
            sCurrentBroadcastIntent.set(data.intent);
            receiver.setPendingResult(data);
            //step 5: 执行onReceive回调
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);
        } catch (Exception e) {
           ...
        } finally {
            sCurrentBroadcastIntent.set(null);
        }

        if (receiver.getPendingResult() != null) {
            data.finish();
        }
    }
```

整个过程:

1. 创建对象LoadedApk;
2. 创建对象BroadcastReceiver;
3. 创建对象Application;
4. 创建对象ContextImpl;
5. 执行onReceive()回调;

说明:

- 以上过程是静态广播接收者, 即通过AndroidManifest.xml的标签来申明的BroadcastReceiver;
- 如果是动态广播接收者,则不需要再创建那么多对象, 因为动态广播的注册时进程已创建, 基本对象已创建完成. 那么只需要回调BroadcastReceiver的onReceive()方法即可.



#### Provider

```java
private IActivityManager.ContentProviderHolder installProvider(Context context, IActivityManager.ContentProviderHolder holder, ProviderInfo info, boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    if (holder == null || holder.provider == null) {
        Context c = null;
        ApplicationInfo ai = info.applicationInfo;
        if (context.getPackageName().equals(ai.packageName)) {
            c = context;
        } else if (mInitialApplication != null &&
                mInitialApplication.getPackageName().equals(ai.packageName)) {
            c = mInitialApplication;
        } else {
            //step 1 && 2: 创建LoadedApk和ContextImpl对象
            c = context.createPackageContext(ai.packageName,Context.CONTEXT_INCLUDE_CODE);
        }

        final java.lang.ClassLoader cl = c.getClassLoader();
        //step 3: 创建ContentProvider对象
        localProvider = (ContentProvider)cl.loadClass(info.name).newInstance();
        provider = localProvider.getIContentProvider();

        //step 4: ContextImpl都attach到ContentProvider对象 [见小节4.4]
        //step 5: 并执行回调onCreate
        localProvider.attachInfo(c, info);
    } else {
        ...
    }
    ...
    return retHolder;
}
```

该方法主要功能:

1. 创建对象LoadedApk;
2. 创建对象ContextImpl;
3. 创建对象ContentProvider;
4. ContextImpl都attach到ContentProvider对象;
5. 执行onCreate回调;

#### Application

```java
private void handleBindApplication(AppBindData data) {
    //step 1: 创建LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    //step 2: 创建ContextImpl对象;
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);

    //step 3: 创建Instrumentation
    mInstrumentation = new Instrumentation();

    //step 4: 创建Application对象; 
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;

    //step 5: 安装providers
    List<ProviderInfo> providers = data.providers;
    installContentProviders(app, providers);

    //step 6: 执行Application.Create回调
    mInstrumentation.callApplicationOnCreate(app);
```

该过程主要功能:

1. 创建对象LoadedApk
2. 创建对象ContextImpl;
3. 创建对象Instrumentation;
4. 创建对象Application;
5. 安装providers;
6. 执行Create回调;



### 核心对象

#### LoadedApk

`frameworks/base/core/java/android/app/LoadedApk.java`

- 是ActivityThread中进行四大组件等启动过程中的重要中间变量
- **LoadedApk对象是APK文件在内存中的表示**。 Apk文件的相关信息，诸如Apk文件的代码和资源，甚至代码里面的Activity，Service等组件的信息我们都可以通过此对象获取。

```java
public final class LoadedApk {

    private static final String TAG = "LoadedApk";

    private final ActivityThread mActivityThread;
    final String mPackageName;
    private ApplicationInfo mApplicationInfo;
    private String mAppDir;
    private String mResDir;
    private String[] mSplitAppDirs;
    private String[] mSplitResDirs;
    private String[] mOverlayDirs;
    private String[] mSharedLibraries;
    private String mDataDir;
    private String mLibDir;
    private File mDataDirFile;
    private File mDeviceProtectedDataDirFile;
    private File mCredentialProtectedDataDirFile;
    private final ClassLoader mBaseClassLoader;
    private final boolean mSecurityViolation;
    private final boolean mIncludeCode;
    private final boolean mRegisterPackage;
    private final DisplayAdjustments mDisplayAdjustments = new DisplayAdjustments();
    /** WARNING: This may change. Don't hold external references to it. */
    Resources mResources;
    private ClassLoader mClassLoader;
    private Application mApplication;

    private final ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>> mReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
    private final ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>> mUnregisteredReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
    private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices
        = new ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>();
    private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mUnboundServices
        = new ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>();

    int mClientCount = 0;

    Application getApplication() {
        return mApplication;
    }

    /**
     * Create information about a new .apk
     *
     * NOTE: This constructor is called with ActivityThread's lock held,
     * so MUST NOT call back out to the activity manager.
     */
    public LoadedApk(ActivityThread activityThread, ApplicationInfo aInfo,
            CompatibilityInfo compatInfo, ClassLoader baseLoader,
            boolean securityViolation, boolean includeCode, boolean registerPackage) {

        mActivityThread = activityThread;
        setApplicationInfo(aInfo);
        mPackageName = aInfo.packageName;
        mBaseClassLoader = baseLoader;
        mSecurityViolation = securityViolation;
        mIncludeCode = includeCode;
        mRegisterPackage = registerPackage;
        mDisplayAdjustments.setCompatibilityInfo(compatInfo);
    }
  ......
}
```



- 重要的成员变量
  - ActivityThread mActivityThread
  - ApplicationInfo mApplicationInfo;
  - String mPackageName;
  - ClassLoader mBaseClassLoader;
  - 以及各种资源路径地址
- 重要方法
  - 生成Application
    - Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation)
  - 生成Resource
    - getResources(ActivityThread mainThread)
    - 实质上，最后是委托ResourceManager去生成的



#### 创建Application

**`LoadedApk.makeApplication`**

```java
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {
    //保证一个LoadedApk对象只创建一个对应的Application对象
    if (mApplication != null) {
        return mApplication;
    }

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application"; //设置应用类名
    }


    java.lang.ClassLoader cl = getClassLoader();
    if (!mPackageName.equals("android")) {
        initializeJavaContextClassLoader();
    }

    //创建ContextImpl对象
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    //创建Application对象, 并将appContext attach到新创建的Application
    Application app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    appContext.setOuterContext(app);
    ...

    mActivityThread.mAllApplications.add(app);
    mApplication = app; //将刚创建的app赋值给mApplication
    ...
    return app;
}
```

```java
static public Application newApplication(Class<?> clazz, Context context) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    Application app = (Application)clazz.newInstance(); //创建Application
    app.attach(context); //执行attach操作
    return app;
}
```

#### 创建ContextImpl

创建ContextImpl的方式有多种, 不同的组件初始化调用不同的方法,如下:

- Activity: 调用createBaseContextForActivity初始化;
- Service/Application: 调用createAppContext初始化;
- Provider: 调用createPackageContext初始化;
- BroadcastReceiver: 直接从Application.getBaseContext()来获取ContextImpl对象;

```java
class ContextImpl extends Context {
    final ActivityThread mMainThread;
    final LoadedApk mPackageInfo;
    private final IBinder mActivityToken;
    private final String mBasePackageName;
    private Context mOuterContext;
    //缓存Binder服务
    final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();

    private ContextImpl(ContextImpl container, ActivityThread mainThread, LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted, Display display, Configuration overrideConfiguration, int createDisplayWithId) {
        mOuterContext = this; //ContextImpl对象
        mMainThread = mainThread; // ActivityThread赋值
        mPackageInfo = packageInfo; // LoadedApk赋值
        mBasePackageName = packageInfo.mPackageName; //mBasePackageName等于“android”
        ...
    }
}
```

### Context attach

#### Activity

`frameworks/base/core/java/android/app/Activity.java`

```java
final void attach(Context context, ActivityThread aThread, Instrumentation instr, IBinder token, int ident, Application application, Intent intent, ActivityInfo info, CharSequence title, Activity parent, String id, NonConfigurationInstances lastNonConfigurationInstances, Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context); //调用父类方法设置mBase.
    mUiThread = Thread.currentThread();
    mMainThread = aThread;
    mApplication = application;
    mIntent = intent;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    ...
}
```

将新创建的ContextImpl赋值到父类ContextWrapper.mBase变量。mApplication赋值。

`frameworks/base/core/java/android/view/ContextThemeWrapper.java`

```java
    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
    }
   
```

`frameworks/base/core/java/android/content/ContextWrapper.java`

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    
      /**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }
```



#### Service

```java
public final void attach( Context context, ActivityThread thread, String className, IBinder token, Application application, Object activityManager) {
    attachBaseContext(context); //调用父类方法设置mBase.
    mClassName = className;
    mToken = token;
    mApplication = application;
    ...
}
```

将新创建的ContextImpl赋值到父类ContextWrapper.mBase变量.

#### BroadcastReceiver

```java
final Context getReceiverRestrictedContext() {
    if (mReceiverRestrictedContext != null) {
        return mReceiverRestrictedContext;
    }
    return mReceiverRestrictedContext = new ReceiverRestrictedContext(getOuterContext());
}
```

对于广播来说Context的传递过程, 跟其他组件完全不同. 广播是在onCreate过程通过参数将ReceiverRestrictedContext传递过去的. 此处getOuterContext()返回的是ContextImpl对象.

#### ContentProvider

```java
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}

private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    mNoPerms = testing;

    if (mContext == null) {
        //将新创建ContextImpl对象保存到ContentProvider对象的成员变量mContext
        mContext = context;
        ...
        if (info != null) {
            setReadPermission(info.readPermission);
            setWritePermission(info.writePermission);
            setPathPermissions(info.pathPermissions);
            mExported = info.exported;
            mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
            setAuthorities(info.authority);
        }
        // 执行onCreate回调;
        ContentProvider.this.onCreate();
    }
}
```

该方法主要功能:

- 将新创建ContextImpl对象保存到ContentProvider对象的成员变量mContext;
  - 可通过getContext()获取该ContextImpl;
- 执行onCreate回调;

#### Application

```java
final void attach(Context context) {
    attachBaseContext(context); //Application的mBase
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}
```

该方法主要功能:

1. 将新创建的ContextImpl对象保存到Application的父类成员变量mBase;
2. 将当前所在的LoadedApk对象保存到Application的父员变量mLoadedApk;



### 总结

每个Apk都对应唯一的application对象和LoadedApk对象, 当Apk中任意组件的创建过程中, 当其所对应的的LoadedApk和Application没有初始化则会创建, 且只会创建一次.

### Context attach过程

1. Application:
   - 调用attachBaseContext()将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
   - 可通过getBaseContext()获取该ContextImpl;
2. Activity/Service:
   - 调用attachBaseContext() 将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
   - 可通过getBaseContext()获取该ContextImpl;
   - 可通过getApplication()获取其所在的Application对象;
3. ContentProvider:
   - 调用attachInfo()将新创建ContextImpl保存到ContentProvider.mContext变量;
   - 可通过getContext()获取该ContextImpl;
4. BroadcastReceiver:
   - 在onCreate过程通过参数将ReceiverRestrictedContext传递过去的.
5. ContextImpl:
   - 可通过getApplicationContext()获取Application;

#### Context使用场景

| 类型        | startActivity | startService | bindService | sendBroadcast | registerReceiver | getContentResolver |
| :---------- | :------------ | :----------- | :---------- | :------------ | :--------------- | :----------------- |
| Activity    | √             | √            | √           | √             | √                | √                  |
| Service     | -             | √            | √           | √             | √                | √                  |
| Receiver    | -             | √            | ×           | √             | -                | √                  |
| Provider    | -             | √            | √           | √             | √                | √                  |
| Application | -             | √            | √           | √             | √                |                    |

说明: (图中第一列代表不同的Context, √代表允许在该Context执行相应的操作; ×代表不允许; -代表分情况讨论)

1. 当Context为Receiver的情况下:
   - 不允许执行bindService()操作, 由于限制性上下文(ReceiverRestrictedContext)所决定的,会直接抛出异常.
   - registerReceiver是否允许取决于receiver;
     - 当receiver == null用于获取sticky广播, 允许使用;
     - 否则不允许使用registerReceiver;
2. 纵向来看startActivity操作
   - 当为Activity Context则可直接使用;
   - 当为其他Context, 则必须带上FLAG_ACTIVITY_NEW_TASK flags才能使用;
   - 另外UI相关要Activity中使用.
3. 除了以上情况, 其他的操作都是被允许执行.

#### getApplicationContext

绝大多数情况下, `getApplication()`和`getApplicationContext()`这两个方法完全一致, 返回值也相同; 那么两者到底有什么区别呢? 真正理解这个问题的人非常少. 接下来彻底地回答下这个问题:

getApplicationContext()这个的存在是Android历史原因. 我们都知道getApplication()只存在于Activity和Service对象; 那么对于BroadcastReceiver和ContentProvider却无法获取Application, 这时就需要一个能在Context上下文直接使用的方法, 那便是getApplicationContext().

两者对比:

1. 对于Activity/Service来说, getApplication()和getApplicationContext()的返回值完全相同; 除非厂商修改过接口;
2. BroadcastReceiver在onReceive的过程, 能使用getBaseContext().getApplicationContext获取所在Application, 而无法使用getApplication;
3. ContentProvider能使用getContext().getApplicationContext()获取所在Application. 绝大多数情况下没有问题, 但是有可能会出现空指针的问题, 情况如下:

当同一个进程有多个apk的情况下, 对于第二个apk是由provider方式拉起的, 前面介绍过provider创建过程并不会初始化所在application, 此时执行 getContext().getApplicationContext()返回的结果便是NULL. 所以对于这种情况要做好判空.



本博客为学习笔记，主要摘抄自 Gityuan [理解Android Context](http://gityuan.com/2017/04/09/android_context/)