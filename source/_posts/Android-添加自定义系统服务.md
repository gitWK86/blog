---
title: Android系统 添加自定义系统服务
date: 2020-08-07 14:47:23
categories: 
- Android系统
tags:
- Android
- Service
- 系统
---

公司准备对设备的Android系统做定制开发，之前都没搞过系统开发，只是了解一些原理性的知识，所以现在就是边做边学习。

系统服务是Android中非常重要的一部分, 像ActivityManagerService（AMS）, PackageManagerSersvice（PMS）, WindowManagerService（WMS）, 这些系统服务都是Framework层的关键服务。本篇文章就了解一下基于Android源码添加一个系统服务的完整流程。



### 编写AIDL文件

文件位置`frameworks/base/core/java/com/myservice/`

**IMyService.aidl**

```java
package android.myservice;

interface IMyService
{
	void setState(int value);
	void setName(String name);
	void setWhiteList(in List<String> list);
}
```

AIDL只支持传输基本java类型数据, 要想传递自定义类, 类需要实现 Parcelable 接口, 另外, 如果传递基本类型数组, 需要指定 in out 关键字, 比如 `void test(in byte[] input, out byte[] output)` , 用 in 还是 out, 只需要记住:  数组如果作为参数, 通过调用端传给被调端, 则使用 in, 如果数组只是用来接受数据, 实际数据是由被调用端来填充的, 则使用 out。

文件写完后, 添加到编译的 Android.mk 中 LOCAL_SRC_FILES 后面:

`frameworks/base/Android.mk`

```
LOCAL_SRC_FILES += \
    core/java/android/accessibilityservice/IAccessibilityServiceConnection.aidl \
	core/java/android/accessibilityservice/IAccessibilityServiceClient.aidl \
	core/java/android/accounts/IAccountManager.aidl \
    部分代码省略 ...
    core/java/com/myservice/IMyService.aidl \
    部分代码省略 ...
```



### Manager类

Android系统中的ManagerService都是不可以直接访问的，需要通过它们的客户端代理类执行操作，我们也为我们的Service写一个代理类。

`frameworks/base/core/java/com/myservice/MyManager.java`



```java
package android.myservice;

import android.myservice.IMyService;
import android.content.Context;
import android.os.RemoteException;
import android.util.Log;
import java.util.List;

public class MyManager {
    

    private final Context mContext;
    private final IMyService mService;

    public MyManager(Context context, IMyService sevice) {
        mContext = context;
        mService = sevice;
    }

    public void setState(int value){
        try {
            mService.setState(value);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
	public void setName(String name){
        try {
            mService.setName(name);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
	public void setWhiteList(List<String> list){
        try {
            mService.setWhiteList(list);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```



### 编写系统服务

`frameworks/base/services/core/java/com/android/server/myservice/MyService.java`

```java
package com.android.server.myservice;


import android.content.Context;
import android.os.Binder;
import android.os.RemoteException;
import android.os.ServiceManager;
import android.util.Log;

import java.util.List;

import android.myservice.MyManager;
import android.myservice.IMyService;


public class MyService extends IMyService.Stub {


    private String TAG = "MyService";
    private Context mContext;

    public MyService(Context context) {
        mContext = context;
    }


    @Override
    public void setState(int value){
        Log.d(TAG,"setState:"+value);
       
    }

    @Override
	public void setName(String name){
        Log.d(TAG,"setName:"+name);
    }

    @Override
	public void setWhiteList(List<String> list){
        for(int i = 0;i<list.size();i++){
            Log.d(TAG,"setWhiteList:"+list.get(i));
        }
        
    }
}
```



### 注册系统服务

在Context中增加系统服务常量

`frameworks/base/core/java/android/content/Context.java`

```java
/** @hide */
    @StringDef({
            POWER_SERVICE,
            WINDOW_SERVICE,
            LAYOUT_INFLATER_SERVICE,
            ...
            MY_SERVICE, //add 
            ...
         
            HARDWARE_PROPERTIES_SERVICE,
            //@hide: SOUND_TRIGGER_SERVICE,
            SHORTCUT_SERVICE,
            //@hide: CONTEXTHUB_SERVICE,
    })
    @Retention(RetentionPolicy.SOURCE)
    public @interface ServiceName {}
```

```java
/**
     * Use with {@link #getSystemService} to retrieve a
     * {@link android.myservice.MyManager}.
     * @hide
     *
     * @see #getSystemService
     * @see android.myservice.MyManager
     */
    public static final String MY_SERVICE = "myservice";
```

所有系统服务都运行在名为 system_server 的进程中, 我们要把编写好的服务加进去, SystemServer中有很多服务, 我们把我们的系统服务加到最后面

`frameworks/base/services/java/com/android/server/SystemServer.java`

```java
import com.android.server.myservice.MyService;

private void startOtherServices() {
     MyService myManagerService = null;
     
     ...
     try{  
          myManagerService = new MyService(context);
		  Log.i("MyService", "In SystemServer, MyService add..");  
		  ServiceManager.addService(Context.MY_SERVICE, myManagerService);  
		} catch (Throwable e) {  
			Log.i("MyService", "In SystemServer, MyService add err.."+e);  
		} 
     
 }
```

这个时候刷机重启，就会报下面的错误，没有权限

```
E/SELinux: avc:  denied  { add } for service=myservice pid=1743 uid=1000 scontext=u:r:system_server:s0 tcontext=u:object_r:default_android_service:s0 tclass=service_manager permissive=0
E/ServiceManager: add_service('myservice',5a) uid=1000 - PERMISSION DENIED
```

添加SELinux权限

`device/qcom/sepolicy/common/service.te`

```
type myservice_service,      app_api_service, system_server_service, service_manager_type;

```

`device/qcom/sepolicy/common/service_contexts`

```
enrichrcsservice                               u:object_r:radio_service:s0
myservice           			               u:object_r:myservice_service:s0
```

权限也可以添加到 `system/sepolicy/service_contexts`和`system/sepolicy/service.te`中，效果相同，但只可以在一个地方添加，否则会报重复。

### 注册Manager

系统服务运行好了, 接下来就是 App获取系统服务, 一般我们都用
`context.getSystemService()`，需要先注册, 代码如下

`frameworks/base/core/java/android/app/SystemServiceRegistry.java`

```java
import android.myservice.MyManager;
import android.myservice.IMyService;
...
registerService(Context.CONTEXTHUB_SERVICE, ContextHubManager.class,
                new CachedServiceFetcher<ContextHubManager>() {
            @Override
            public ContextHubManager createService(ContextImpl ctx) {
                return new ContextHubManager(ctx.getOuterContext(),
                  ctx.mMainThread.getHandler().getLooper());
            }});

// add MyManager
registerService(Context.MY_SERVICE, MyManager.class,
			new CachedServiceFetcher<MyManager>() {
				@Override
				public MyManager createService(ContextImpl ctx) {
					IBinder b = ServiceManager.getService(Context.MY_SERVICE);
					IMyService service = IMyService.Stub.asInterface(b);
					return new MyManager(ctx, service);
		}});
```

OK, 系统代码修改完成了!



编译系统（可以单独编译boot.img和system.img，权限控制在boot.img，代码修改在system.img）、刷机

系统启动后

```
1970-01-13 04:01:02.700 1550-1550/system_process I/MyService: In SystemServer, MyService add..
```

没有报错，表示启动成功！



### App调用

将系统编译出的class.jar导入到工程中，否则找不到对应的Service（请自行百度），然后使用context.getSystemService("myservice")获取MyManager对象，最终调用到MyService中的方法。

客户端App：

![](Android-添加自定义系统服务/1.png)

Service：

![](Android-添加自定义系统服务/2.png)

搞定！