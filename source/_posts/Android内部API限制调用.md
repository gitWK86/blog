---
title: Android内部API限制调用
date: 2021-08-10 13:18:10
tags:
---

最近开发过程中遇到一个问题，在APP中有一个功能想要反射调用ActivityTaskManager中的函数（系统中增加的接口）。ActivityTaskmanager是一个hide类，通过SDK无法调用。

```java
/**
 * This class gives information about, and interacts with activities and their containers like task,
 * stacks, and displays.
 *
 * @hide
 */
@TestApi
@SystemService(Context.ACTIVITY_TASK_SERVICE)
public class ActivityTaskManager{
    ActivityTaskManager(Context context, Handler handler) {
    }
}
```

因此想使用反射去调用其中的函数。

然后就发现，明明有构造函数，但是却无法通过`getDeclaredConstructor`等相关的获取构造函数的接口获取到，也就无法构建对象，就感觉很奇怪，因此查了一下原因。

引用Android开发者平台原文：

> 从 Android 9（API 级别 28）开始，Android 平台对应用能使用的非 SDK 接口实施了限制。只要应用引用非 SDK 接口或尝试使用反射或 JNI 来获取其句柄，这些限制就适用。这些限制旨在帮助提升用户体验和开发者体验，为用户降低应用发生崩溃的风险，同时为开发者降低紧急发布的风险。

也就是说从Android 9开始就对反射调用内部API做了限制，所以很有可能ActivityTaskManager就在被限制范围内。

### 非 SDK API 名单

为最大程度地降低非 SDK 使用限制对开发工作流的影响，Android将非 SDK 接口分成了几个名单，这些名单界定了非 SDK 接口使用限制的严格程度（取决于应用的目标 API 级别）。下表介绍了这些名单：

| 名单                          | 说明                                                         |
| :---------------------------- | :----------------------------------------------------------- |
| 屏蔽名单 (`blacklist`)        | 无论应用的目标 API 级别是什么，您都无法使用的非 SDK 接口。 如果您的应用尝试访问其中任何一个接口，系统就会抛出错误。 |
| 有条件屏蔽 (`greylist-max-x`) | 从 Android 9（API 级别 28）开始，当有应用以该 API 级别为目标平台时，我们会在每个 API 级别分别限制某些非 SDK 接口。这些名单会以应用无法再访问该名单中的非 SDK 接口之前可以作为目标平台的最高 API 级别 (`max-target-x`) 进行标记。例如，在 Android Pie 中未被屏蔽、但现在已被 Android 10 屏蔽的非 SDK 接口会列入 `max-target-p` (`greylist-max-p`) 名单，其中的“p”表示 Pie 或 Android 9（API 级别 28）。如果您的应用尝试访问受目标 API 级别限制的接口，系统就会将此 API 视为已列入屏蔽名单。 |
| 不支持 (`greylist`)           | 当前不受限制且您的应用可以使用的非 SDK 接口。 但请注意，这些接口**不受支持**，可能会在不另行通知的情况下随时发生更改。预计这些接口在未来的 Android 版本中会被有条件地屏蔽，并列在 `max-target-x` 名单中。 |
| SDK (`whitelist`)             | 已在 Android 框架软件包索引中正式记录、受支持并且可以自由使用的接口。 |

屏蔽名单 (`blacklist`) 和有条件屏蔽的 API 名单（深灰名单）是在构建时派生的。我们可以通过命令：

```
m out/soong/hiddenapi/hiddenapi-flags.csv
```

然后，您便可以在以下位置找到该文件：

```
out/soong/hiddenapi/hiddenapi-flags.csv
```



![hiddenapi-flags.csv](Android内部API限制调用机制\blacklist.png)

可以看到ActivityTaskManager的构造函数和接口基本都是在黑名单中，无法通过反射调用。因此通过反射调用ActivityTaskManager接口的方式宣告失败。只能将APK放在源码环境下编译解决。



### 访问受限的非 SDK 接口时可能会出现的预期行为

下表说明了当您的应用尝试访问屏蔽名单中的非 SDK 接口时可能会出现的预期行为。

| 访问方式                                                     | 结果                                  |
| :----------------------------------------------------------- | :------------------------------------ |
| Dalvik 指令引用某个字段                                      | 抛出 `NoSuchFieldError`               |
| Dalvik 指令引用某个方法                                      | 抛出 `NoSuchMethodError`              |
| 通过 `Class.getDeclaredField()` 或 `Class.getField()` 进行反射 | 抛出 `NoSuchFieldException`           |
| 通过 `Class.getDeclaredMethod()`、`Class.getMethod()` 进行反射 | 抛出 `NoSuchMethodException`          |
| 通过 `Class.getDeclaredFields()`、`Class.getFields()` 进行反射 | 结果中未获取到非 SDK 成员             |
| 通过 `Class.getDeclaredMethods()`、`Class.getMethods()` 进行反射 | 结果中未获取到非 SDK 成员             |
| 通过 `env->GetFieldID()` 进行 JNI 调用                       | 返回 `NULL`，抛出 `NoSuchFieldError`  |
| 通过 `env->GetMethodID()` 进行 JNI 调用                      | 返回 `NULL`，抛出 `NoSuchMethodError` |

