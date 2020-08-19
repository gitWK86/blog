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

目前处于 Alpha 版测试阶段，因为其 API 界面尚未最终确定。我们不建议在生产环境中使用 Alpha 库。CameraX 库应在生产环境中严格避免依赖 Alpha 库，因为其 API 界面可能会以与源代码和二进制文件不兼容的方式发生变化。

相比较于使用Camera2预览、拍照时大量的接口、回调，使用CameraX基本可以使用不超过100行代码实现相同功能。虽然目前仍是测试版本，但个人强烈建议先学习下，CameraX 真的超简单，超好用！！！后面正式版发布后就可以随时使用。



## CameraX使用

### CameraX 结构

开发者使用 CameraX，借助名为“用例”的抽象概念与设备的相机进行交互。目前提供的用例如下：

- 预览：准备一个预览 SurfaceTexture
- 图片拍摄：拍摄并保存照片
- 图片分析：提供 CPU 可访问的缓冲区以进行分析（例如进行机器学习）

不同用例可以相互组合使用，也可以同时处于活动状态。例如，用户可以在应用中使用预览用例查看进入相机视野的画面、加入图片分析用例来确定照片里的人物是否在微笑，以及包含一个图片拍摄用例以便在人物微笑时拍摄照片。

### 添加依赖

```
    def camerax_version = "1.0.0-beta04"
     // CameraX core library using camera2 implementation
    implementation "androidx.camera:camera-camera2:$camerax_version"
     // CameraX Lifecycle Library
    implementation "androidx.camera:camera-lifecycle:$camerax_version"
     // CameraX View class
    implementation "androidx.camera:camera-view:1.0.0-alpha11"
    
```

### 布局

```xml
<androidx.camera.view.PreviewView
        android:id="@+id/viewFinder"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
```

使用androidx.camera.view.PreviewView类。它是CameraX中显示预览用例的自定义视图。该类管理Surface生命周期，以及预览纵横比和方向。在它内部使用TextureView或SurfaceView来显示。



### 实现预览

```kotlin
private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener(Runnable {
            // Used to bind the lifecycle of cameras to the lifecycle owner
            val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()

            // Preview
            preview = Preview.Builder()
                    .build()

            // Select back camera
            val cameraSelector = CameraSelector.Builder().requireLensFacing(CameraSelector.LENS_FACING_BACK).build()

            try {
                // Unbind use cases before rebinding
                cameraProvider.unbindAll()

                // Bind use cases to camera
                camera = cameraProvider.bindToLifecycle(
                        this, cameraSelector, preview)
                preview?.setSurfaceProvider(viewFinder.createSurfaceProvider())
            } catch (exc: Exception) {
                Log.e(TAG, "Use case binding failed", exc)
            }

        }, ContextCompat.getMainExecutor(this))
    }
```

就这样就可以实现Camera的预览功能，是不是很简单，想起之前写Camera2的痛苦，眼泪都快流下来了。



### 拍照

```kotlin
//在startCamera中增加
// ImageCapture
imageCapture = ImageCapture.Builder()
               .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
               .build()
```



```kotlin

private fun takePhoto() {
    // Get a stable reference of the modifiable image capture use case
    val imageCapture = imageCapture ?: return

    // Create timestamped output file to hold the image
    val photoFile = File(
            outputDirectory,
            SimpleDateFormat(FILENAME_FORMAT, Locale.CHINA
            ).format(System.currentTimeMillis()) + ".jpg")

    // Create output options object which contains file + metadata
    val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

    // Setup image capture listener which is triggered after photo has
    // been taken
    imageCapture.takePicture(
            outputOptions, ContextCompat.getMainExecutor(this), object : ImageCapture.OnImageSavedCallback {
        override fun onError(exc: ImageCaptureException) {
            Log.e(TAG, "Photo capture failed: ${exc.message}", exc)
        }

        override fun onImageSaved(output: ImageCapture.OutputFileResults) {
            val savedUri = Uri.fromFile(photoFile)
            val msg = "Photo capture succeeded: $savedUri"
            Toast.makeText(baseContext, msg, Toast.LENGTH_SHORT).show()
            Log.d(TAG, msg)
        }
    })
}
```

```kotlin
//最后绑定到Camera上
// Bind use cases to camera
camera = cameraProvider.bindToLifecycle(
          this, cameraSelector, preview,imageCapture)
```

完成了，这代码简洁程度简直爱了！



### 图片分析

写一个内部类，继承ImageAnalysis.Analyzer 

```kotlin
private class LuminosityAnalyzer(private val listener: LumaListener) : ImageAnalysis.Analyzer {

    private fun ByteBuffer.toByteArray(): ByteArray {
        rewind()    // Rewind the buffer to zero
        val data = ByteArray(remaining())
        get(data)   // Copy the buffer into a byte array
        return data // Return the byte array
    }

    override fun analyze(image: ImageProxy) {
        //处理图片数据
        val buffer = image.planes[0].buffer
        val data = buffer.toByteArray()
        val pixels = data.map { it.toInt() and 0xFF }
        val luma = pixels.average()
        listener(luma)
        image.close()
    }
}
```

```
imageAnalyzer = ImageAnalysis.Builder()
        .build()
        .also {
            it.setAnalyzer(cameraExecutor, LuminosityAnalyzer { luma ->
                Log.d(TAG, "Average luminosity: $luma")
            })
        }
```

绑定设备

```kotlin
// Bind use cases to camera
camera = cameraProvider.bindToLifecycle(
        this, cameraSelector, preview, imageCapture, imageAnalyzer)
```

仍然是这么简单！等CameraX正式版本发布，Camera2就扔到垃圾桶去吧。



记得申请权限啊！`Manifest.permission.CAMERA`

可以翻墙的请看原文：[Getting Started with CameraX](https://codelabs.developers.google.com/codelabs/camerax-getting-started/#0)