---
title: ResourceManager广播管理功能重构设计文档
date: 2021-07-27 15:57:45
tags:
---



## 架构图

![](ResourceManager广播管理功能重构设计文档\broadcastManager架构.png)

## BroadCastQueue

首先梳理一下Android原生的`BroadCastQueue`中的处理逻辑。

### 构造函数

```java
BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, BroadcastConstants constants, boolean allowDelayBehindServices) {
        mService = service;
        mHandler = new BroadcastHandler(handler.getLooper());
        mQueueName = name;
        mDelayBehindServices = allowDelayBehindServices;
        mConstants = constants;
        mDispatcher = new BroadcastDispatcher(this, mConstants, mHandler, mService);
        
        mTclBroadcastQueue = TclServiceFactory.getTclBroadcastQueue(service, this);
    }
```

