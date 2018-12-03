---
title: Android JNI学习-LOG日志
date: 2018-12-03 10:43:38
categories: 
- Android开发
tags:
- JNI
- NDK
---

刚好在准备一个有JNI开发的项目，就想着先用Demo练下手，毕竟好久没做过了。做的时候发现自己忘记了Log信息怎么打印的，就网上搜索了下，结果一堆让修改Android.mk的，这些都是以前eclipse或者旧版本AS的用法，所以在这里记录一下AS上JNI Log的使用，方便以后查看使用。

### 修改build.gradle

    defaultConfig {
       ndk {
           ldLibs "log" //实现__android_log_print
           moduleName "demo"  //设置库(so)文件名称
           abiFilters  "armeabi-v7a", "x86"
       }
    }
`ldLibs "log" `是实现JNI Log的关键。



### 引入

```c
#include <android/log.h>

#define LOG_TAG    "MyDemo"
#define LOGI(...)  __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...)  __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
#define LOGD(...)  __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
```



### 使用

```c
LOGE("myName : %s", name); // Log.e(TAG,"myName : $name")
LOGD("age: %d",age); // Log.d(TAG,"age : $age")
```

