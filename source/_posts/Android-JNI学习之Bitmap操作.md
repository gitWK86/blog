---
title: Android JNI学习之Bitmap操作
date: 2018-08-30 08:46:32
tags:
---

之前有学习了Android JNI的基本操作字符串操作、数组操作以及C++调用Java层代码，今天就来看一下JNI操作Bitmap。这个也是JNI常用的功能之一，很多图像处理应用也都是通过这种方式来对图片进行各种色彩、像素处理，要比在Java中处理速度高很多。

> C++水平还不够，如有错误欢迎指出

## Bitmap.h

有关Bitmap的操作都定义在了bitmap.h头文件中，内容并不多，主要有这些部分组成：

* 位图操作函数声明。
* 结果状态定义。
* 位图格式枚举。
* 位图信息结构体。

### 1. 位图操作函数声明

```
int AndroidBitmap_getInfo(JNIEnv* env, jobject jbitmap,
                          AndroidBitmapInfo* info);
                          
int AndroidBitmap_lockPixels(JNIEnv* env, jobject jbitmap, void** addrPtr);

int AndroidBitmap_unlockPixels(JNIEnv* env, jobject jbitmap);
```

* `AndroidBitmap_getInfo`：获取当前位图信息。
* `AndroidBitmap_lockPixels`：锁定当前位图像素，在锁定期间该Bitmap对象不会被回收，使用完成之后必须调用`AndroidBitmap_unlockPixels`函数来解除对像素的锁定。
* `AndroidBitmap_unlockPixels`：解除像素锁定。

### 2. 结果状态定义

```
enum {
    /** Operation was successful. */
    ANDROID_BITMAP_RESULT_SUCCESS           = 0, //成功
    /** Bad parameter. */
    ANDROID_BITMAP_RESULT_BAD_PARAMETER     = -1, //错误的参数
    /** JNI exception occured. */
    ANDROID_BITMAP_RESULT_JNI_EXCEPTION     = -2, //JNI异常
    /** Allocation failed. */
    ANDROID_BITMAP_RESULT_ALLOCATION_FAILED = -3, //内存分配错误
};
```

### 3. 位图格式枚举

```
/** Bitmap pixel format. */
enum AndroidBitmapFormat {
    /** No format. */
    ANDROID_BITMAP_FORMAT_NONE      = 0,
    /** Red: 8 bits, Green: 8 bits, Blue: 8 bits, Alpha: 8 bits. **/
    ANDROID_BITMAP_FORMAT_RGBA_8888 = 1,
    /** Red: 5 bits, Green: 6 bits, Blue: 5 bits. **/
    ANDROID_BITMAP_FORMAT_RGB_565   = 4,
    /** Deprecated in API level 13. Because of the poor quality of this configuration, it is advised to use ARGB_8888 instead. **/
    ANDROID_BITMAP_FORMAT_RGBA_4444 = 7,
    /** Alpha: 8 bits. */
    ANDROID_BITMAP_FORMAT_A_8       = 8,
};
```

从代码中可以看到每种格式对应的像素长度不同，`ARGB_8888`占32位，`RGB_565`占16位，选用`ARGB_8888`所能够表示的颜色要比`RGB_565`更多，但是却需要占用更多的内存空间。

### 4. 位图信息结构体
```
/** Bitmap info, see AndroidBitmap_getInfo(). */
typedef struct {
    /** The bitmap width in pixels. */
    uint32_t    width; //图片宽度（列数）
    /** The bitmap height in pixels. */
    uint32_t    height; //图片高度(行数)
    /** The number of byte per row. */
    uint32_t    stride; //行跨度
    /** The bitmap pixel format. See {@link AndroidBitmapFormat} */
    int32_t     format;
    /** Unused. */
    uint32_t    flags;      // 0 for now
} AndroidBitmapInfo;
```

## JNI操作BitMap例子

