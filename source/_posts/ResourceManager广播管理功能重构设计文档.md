---
title: ResourceManager广播管理功能重构设计文档
date: 2021-07-27 15:57:45
tags:
---

在`ResourceManager`中有一个负责管理广播的功能模块，主要功能是缓存`resource_policy.conf`中解析出的配置信息，对外提供广播白名单、黑名单以及广播发送是否跳过应用等功能。

## 原架构

![](ResourceManager广播管理功能重构设计文档\原broadcastManager架构.png)

## 原主要逻辑代码

```java
public final class ResourceBroadcast {
    public ResourceBroadcast(boolean enable, ArrayList<String> appblacklist,
                                 ArrayList<String> appwhitelist, ArrayList<String> skipnamelist,
                                 ArrayList<String> ctswhiteactionlist,
                                 ArrayList<ActionInfo> skipActionInfo) {
            this.enable = enable;
            this.mAppblacklist = appblacklist;
            this.mAppwhitelist = appwhitelist;
            this.mSkipnamelist = skipnamelist;
            this.mCtsWhiteActionList = ctswhiteactionlist;
            this.mSkipActionInfo = skipActionInfo;
            mSkipActions = new ArrayList<String>();
            mIsPMode = isTclFactoryPMode();
        }


    public boolean isSkipActionAndPackage(String actionName, String packageName) {
            if (!enable) {
                return false;
            }

            if (!mSkipActions.contains(actionName)) {
                return false;
            }

            for (ActionInfo actionInfo : mSkipActionInfo) {
                if (actionInfo.isSkipActionAndPackage(actionName, packageName, mIsPMode)) {
                    return true;
                }
            }
            return false;
        }

    private boolean isTclFactoryPMode() {
          File f = new File("/userdata/sita_P");
          return f.exists();
    }
}
```



可以看到，原本的逻辑中，对`resource_policy.conf`文件进行解析后，将广播相关的数据保存在`ResourceBroadcast`对象中。同时，又将模式判断、是否跳过广播等逻辑也在`ResourceBroadcast`中完成处理。将业务逻辑和数据耦合在一个类中明显是不太合理的处理，因此对广播管理功能进行重构。



## 重构后架构



![](ResourceManager广播管理功能重构设计文档\broadcastManager架构.png)

重构后：

1. 增加`BroadcastManager`管理类，对广播功能进行封装，对外提供统一接口
2. 增加`TVManagerUtil`工具类，判断设备当前模式(P模式、D模式或酒店模式)
3. `ResourceBroadcast`作为数据类仅保留解析数据缓存功能



## 重构后时序图



![](ResourceManager广播管理功能重构设计文档\ResourceManager-Broadcast.png)

## 重构后代码

### BroadcastManager

目前对外提供功能：功能开关判断`isBcEnable()`、获取黑白名单列表`getBcList()`、广播是否跳过`isSkipActionAndPackage()`

```java
public class BroadcastManager {

    private static final String TAG = "BroadcastManager";
    private Context mContext;
    private ResourceBroadcast mResourceBroadcast;
    private ArrayList<ActionInfo> mSkipActionInfoList;
    private ArrayList<String> mSkipActionNameList;
    private boolean mBcEnable = false;
    private boolean mIsFactoryOrHotelMode = false;

    public BroadcastManager(Context context, ResourcePolicyConfig config) {
        mContext = context;
        if (config != null) {
            mResourceBroadcast = config.getBroadcastSettings();
            initSkipInfo();
        }
    }

    // must be invoked in RMS systemReady, and becareful tvmanager is down.
    public void systemReady() {
        Slog.d(TAG,"systemReady");
        TVManagerUtil tvManagerUtil = TVManagerUtil.getInstance(mContext);
        if (tvManagerUtil != null) {
            mIsFactoryOrHotelMode = tvManagerUtil.isFactoryOrHotelMode();
            Slog.d(TAG,"isFactoryOrHotelMode:" + mIsFactoryOrHotelMode);
        }
    }

    public boolean isBcEnable() {
        return mBcEnable;
    }

    public List getBcList(String key) {
        List<String> res = new ArrayList<String>();
        if (mResourceBroadcast != null) {
            if (key.trim().equals("bl")) {
                res = mResourceBroadcast.getBlacklist();
            } else if (key.trim().equals("wl")) {
                res = mResourceBroadcast.getWhitelist();
            } else if ((key.trim().equals("nl"))) {
                res = mResourceBroadcast.skipNamelist();
            } else if (key.trim().equals("ctswl")) {
                res = mResourceBroadcast.getCtsWhiteActionList();
            }
        }
        return res;
    }

    public boolean isSkipActionAndPackage(String actionName, String packageName) {
        if (mResourceBroadcast == null) {
            return false;
        }

        if (!mBcEnable) {
            return false;
        }

        if (!mSkipActionNameList.contains(actionName)) {
            return false;
        }

        for (ActionInfo actionInfo : mSkipActionInfoList) {
            if (mIsFactoryOrHotelMode) {
                return actionInfo.getActionName().equals(actionName) && actionInfo.getPModePackageNames().contains(packageName);
            } else {
                return actionInfo.getActionName().equals(actionName) && actionInfo.getPackageNames().contains(packageName);
            }
        }

        return false;
    }

    private void initSkipInfo() {
        if (mResourceBroadcast != null) {
            mSkipActionInfoList = mResourceBroadcast.getSkipActionInfo();
            mSkipActionNameList = mResourceBroadcast.getSkipActionNames();
            mBcEnable = mResourceBroadcast.getEnable();
        }
    }
}

```

### TVManagerUtil

主要功能是判断P模式、D模式和酒店模式

```java
public class TVManagerUtil {
    private static final String TAG = "TVManagerUtil";
    private volatile static TVManagerUtil sInstance = null;
    private FactoryManager mFactoryManager = null;
    private boolean mIsFactoryOrHotelMode = false;

    private TVManagerUtil(Context context) {
        mFactoryManager = FactoryManager.getInstance(context);
    }

    public static TVManagerUtil getInstance(Context context) {
        if (sInstance == null) {
            synchronized (TVManagerUtil.class) {
                if (sInstance == null) {
                    sInstance = new TVManagerUtil(context);
                }
            }
        }
        return sInstance;
    }

    public boolean isFactoryOrHotelMode() {
        boolean isPMode = isTclFactoryPMode();
        boolean isDMode = isTclFactoryDMode();
        boolean isHotelMode = isTclHotelMode();
        Slog.d(TAG, "PModeEnable:" + isPMode + " DModeEnable:" + isDMode 
                + " HotelModeEnable:" + isHotelMode);
        return isPMode || isDMode || isHotelMode;
    }

    private boolean isTclFactoryPMode() {
        File f = new File("/userdata/sita_P");
        return f.exists();
    }

    private boolean isTclHotelMode() {
        return SystemProperties.getInt("persist.tcl.hotel.enable", 0) == 1;
    }

    private boolean isTclFactoryDMode() {
        if (mFactoryManager == null) {
            return false;
        }
        return mFactoryManager.doGetAttribute(FactoryManager.FACTORY_ATTR_D_MODE) != 0;
    }
}
```

### ResourceBroadcast

作为数据类，只提供数据的`get`和`set`接口

```java
public final class ResourceBroadcast {

    private ArrayList<String> mAppblacklist;

    private ArrayList<String> mAppwhitelist;

    private ArrayList<String> mSkipnamelist;

    private ArrayList<String> mCtsWhiteActionList;

    private ArrayList<ActionInfo> mSkipActionInfo;

    private ArrayList<String> mSkipActions;

    private boolean enable;
    private final static String TAG = "ResourceManagerService";

    public ResourceBroadcast() {
        this(true, new ArrayList<String>(), new ArrayList<String>(),
                new ArrayList<String>(), new ArrayList<String>(), new ArrayList<ActionInfo>());
    }

    public ResourceBroadcast(boolean enable, ArrayList<String> appblacklist,
                             ArrayList<String> appwhitelist, ArrayList<String> skipnamelist,
                             ArrayList<String> ctswhiteactionlist,
                             ArrayList<ActionInfo> skipActionInfo) {
        this.enable = enable;
        this.mAppblacklist = appblacklist;
        this.mAppwhitelist = appwhitelist;
        this.mSkipnamelist = skipnamelist;
        this.mCtsWhiteActionList = ctswhiteactionlist;
        this.mSkipActionInfo = skipActionInfo;
        mSkipActions = new ArrayList<String>();
    }


    //add by jxp
    public void setEnable(boolean enable) {
        this.enable = enable;
    }

    public void setBlacklist(ArrayList<String> blacklist) {
        this.mAppblacklist = blacklist;
    }

    public void setWhiteList(ArrayList<String> WhiteList) {
        this.mAppwhitelist = WhiteList;
    }

    public void skipNamelist(ArrayList<String> skipNamelist) {
        this.mSkipnamelist = skipNamelist;
    }

    public boolean getEnable() {
        return enable;
    }

    public ArrayList<String> getBlacklist() {
        return mAppblacklist;
    }

    public ArrayList<String> getWhitelist() {
        return mAppwhitelist;
    }

    public ArrayList<String> skipNamelist() {
        return mSkipnamelist;
    }

    public void setCtsWhiteActionList(ArrayList<String> ctsWhiteActionList) {
        this.mCtsWhiteActionList = ctsWhiteActionList;
    }

    public ArrayList<String> getCtsWhiteActionList() {
        return mCtsWhiteActionList;
    }

    public ArrayList<ActionInfo> getSkipActionInfo() {
        return mSkipActionInfo;
    }

    public ArrayList<String> getSkipActionNames() {
        return mSkipActions;
    }

    public void SetActionInfo(ActionInfo actionInfo) {
        mSkipActionInfo.add(actionInfo);
        mSkipActions.add(actionInfo.getActionName());
    }


    public static class ActionInfo {
        private String actionName;
        private ArrayList<String> packageNames;
        private ArrayList<String> pModePackageNames;

        public ActionInfo() {
            actionName = "";
            packageNames = new ArrayList<>();
            pModePackageNames = new ArrayList<>();
        }

        public ActionInfo(String actionName, ArrayList<String> packageNames,
                          ArrayList<String> pModePackageNames) {
            this.actionName = actionName;
            this.packageNames = packageNames;
            this.pModePackageNames = pModePackageNames;
        }

        public String getActionName() {
            return actionName;
        }

        public void setActionName(String actionName) {
            this.actionName = actionName;
        }

        public ArrayList<String> getPackageNames() {
            return packageNames;
        }

        public void setPackageNames(ArrayList<String> packageNames) {
            this.packageNames = packageNames;
        }

        public ArrayList<String> getPModePackageNames() {
            return pModePackageNames;
        }

        public void setPModePackageNames(ArrayList<String> pModePackageNames) {
            this.pModePackageNames = pModePackageNames;
        }


        @Override
        public String toString() {
            return "ActionInfo{" +
                    "actionName='" + actionName + '\'' +
                    ", packageNames=" + packageNames +
                    ", pModePackageNames=" + pModePackageNames +
                    '}';
        }
    }
}
```



## 广播发送完整流程

![](ResourceManager广播管理功能重构设计文档\broadcast.png)

上图是广播从`sendBroadcast`发送到`onReceive`接收的完整时序图。

可以看到主要的处理逻辑是在`BroadcastQueue`中。在`BroadcastQueue`中调用`TclBroadcastQueueImpl`的`skipSpecialBroadcast()`函数，然后`TclBroadcastQueueImpl`中调用`TclBroadcastManager`的`skipSpecialBroadcast()`，再通过`ResourceManager`提供的对外接口`IActivityListener`最终调用到`ResourceManager`内部去处理广播逻辑。



