---
title: Android开发-Jetpack组件CameraX
date: 2020-02-25 11:36:26
categories: 
- Android开发
tags:
- CameraX
---

CameraX 是一个 Jetpack 支持库，旨在帮助您简化相机应用的开发工作。它提供一致且易于使用的 API 界面，适用于大多数 Android 设备，并可向后兼容至 Android 5.0（API 级别 21）。

虽然它利用的是 camera2 的功能，但使用的是更为简单且基于用例的方法，该方法具有生命周期感知能力。它还解决了设备兼容性问题，因此您无需在代码库中包含设备专属代码。这些功能减少了将相机功能添加到应用时需要编写的代码量。

最后，借助 CameraX，开发者只需两行代码就能利用与预安装的相机应用相同的相机体验和功能。 [CameraX Extensions](https://developer.android.google.cn/training/camerax/vendor-extensions) 是可选插件，通过该插件，您可以在[支持的设备](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-master-dev/camera/camera-extensions/ExtensionsSupportedDevices.md)上向自己的应用中添加人像、HDR、夜间模式和美颜等效果。





## CameraX使用

> [CameraX 库](https://developer.android.google.cn/reference/androidx/camera/core/package-summary)目前处于 Alpha 版测试阶段，因为其 API 界面尚未最终确定。我们不建议在生产环境中使用 Alpha 库。CameraX 库应在生产环境中严格避免依赖 Alpha 库，因为其 API 界面可能会以与源代码和二进制文件不兼容的方式发生变化。

### CameraX 结构

开发者使用 CameraX，借助名为“用例”的抽象概念与设备的相机进行交互。目前提供的用例如下：

- 预览：准备一个预览 SurfaceTexture
- 图片分析：提供 CPU 可访问的缓冲区以进行分析（例如进行机器学习）
- 图片拍摄：拍摄并保存照片

不同用例可以相互组合使用，也可以同时处于活动状态。例如，用户可以在应用中使用预览用例查看进入相机视野的画面、加入图片分析用例来确定照片里的人物是否在微笑，以及包含一个图片拍摄用例以便在人物微笑时拍摄照片。

### 添加依赖

```
    dependencies {
        // CameraX core library.
        def camerax_version = "1.0.0-alpha05"
        implementation "androidx.camera:camera-core:${camerax_version}"
        // If you want to use Camera2 extensions.
        implementation "androidx.camera:camera-camera2:${camerax_version}"
    }
    
```

### 布局

```xml
<androidx.camera.view.PreviewView
    android:id="@+id/view_finder"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

使用androidx.camera.view.PreviewView类。它是CameraX中显示预览用例的自定义视图。该类管理Surface生命周期，以及预览纵横比和方向。在它内部使用TextureView或SurfaceView来显示。



### 实现预览



