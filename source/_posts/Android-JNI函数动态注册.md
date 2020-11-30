---
title: Android JNI学习-函数动态注册
date: 2020-11-09 09:46:15
categories: 
- Android开发
tags:
- JNI
- NDK
---

前面JNI开发相关的也写了几篇博客，对java中native关键字定义的方法进行注册时，都是使用Javah命令生成对应的`Java _ 包名 _ 类名 _ 方法名`，现在完全可以通过编译器帮我们生成，我们去填对应的逻辑代码即可，这种方式被称为**静态注册**。今天来看一下新的方式：**动态注册**

不同于静态注册中在Java类中定义好native方法后由编译器生成JNI方法，动态注册基本思想是在JNI_Onload()函数中通过JNI中提供的RegisterNatives()方法来将C/C++方法和java方法对应起来, JNI_OnLoad ()函数会在我们调用 System.loadLibrary的时候回调，注册整体流程如下:

1. 定义Java类中的native方法
2. 编写C/C++代码, 实现JNI_Onload()方法
3. 将Java 方法和 C/C++方法通过签名信息对应起来
4. 通过JavaVM获取JNIEnv, JNIEnv主要用于获取Java类和调用一些JNI提供的方法
5. 使用类名和对应起来的方法作为参数, 调用JNI提供的函数RegisterNatives()注册方法



### 1、Java Native方法

```java
public native String getStringFromC();

public native int getIntFromC(int index);
```



### 2、C/C++方法

```c
jstring returnString(JNIEnv *env, jobject instance) {
    char *str = "I come from C＋＋";
    return env->NewStringUTF(str);
}

jint returnInt(JNIEnv *env, jobject instance, jint index) {
    return index + 10;
}
```



 ### 3、JNINativeMethod

```
static JNINativeMethod gMethods[] = {
        {"getStringFromC", "()Ljava/lang/String;", (void *) returnString},
        {"getIntFromC",    "(I)I",                 (void *) returnInt}
};
```

- 第一个参数对应的native方法名
- 第二个参数对应 native方法的描述
- 第三个参数对应的c++代码里对应的实现

通过这个数组将Java层函数和C/C++层代码对应起来

### 4、JNI_Onload

```c
int JNI_OnLoad(JavaVM *vm, void *re) {
    JNIEnv *env;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    jclass javaClass = env->FindClass("com/david/jnitestdemo/MainActivity");
    if (javaClass == NULL) {
        return JNI_ERR;
    }
    if (env->RegisterNatives(javaClass, gMethods, sizeof(gMethods) / sizeof(gMethods[0])) < 0) {
        return JNI_ERR;
    }
    return JNI_VERSION_1_6;
}
```

`env->RegisterNatives(javaClass, gMethods, sizeof(gMethods) / sizeof(gMethods[0])`

第一个表示对应jclass，第二个表示JNINativeMethod的数组，第三个是函数的数量



这样就完成了简单的JNI动态注册Demo

对比下之前的静态注册：

```c
//静态注册
extern "C" JNIEXPORT jstring JNICALL
Java_com_david_jnitestdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
}
 
//动态注册
jstring returnString(JNIEnv *env, jobject instance) {
}
```

相比较来说动态注册的代码会清爽一些，虽然多了JNI_OnLoad和JNINativeMethod，但是JNI_OnLoad基本可以只写一次，JNINativeMethod每次有新增函数时才修改，所以个人感觉动态注册写代码会更舒服些，也看个人习惯 ,好了就到这里了。 ^-^



