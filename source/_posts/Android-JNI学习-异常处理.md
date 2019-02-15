---
title: Android JNI学习-异常处理
date: 2019-02-14 21:25:55
categories: 
- Android开发
tags:
- JNI
- NDK
---

异常我们已经很熟悉了，空指针、数组越界等等，在Java中，当抛出一个异常，虚拟机会停止执行代码块并进入调用栈反向检查能处理特定异常的异常处理程序代码块，虚拟机清除异常并将控制权交给异常处理程序。而JNI不同，JNI没有像Java一样有try…catch…final这样的异常处理机制，面且在本地代码中调用某个JNI接口时如果发生了异常，后续的本地代码不会立即停止执行，而会继续往下执行后面的代码，这就要求开发人员在异常发生后显式地实现异常处理。

### 1 捕获异常

在一个方法执行之后，可以调用`(*env)->ExceptionCheck`或者 `(*env)->ExceptionOccurred`（两者的区别在于返回值不一样）

`ExceptionCheck`：检查是否发生了异常，若有异常返回JNI_TRUE，否则返回JNI_FALSE  `ExceptionOccurred`：检查是否发生了异常，若用异常返回该异常的引用，否则返回NULL 

```
     const jchar *cstr = (*env)->GetStringChars(env, jstr);
     if (c_str == NULL) {
         return; 
     }
     if ((*env)->ExceptionCheck(env)) { /* 异常检查 */
         (*env)->ReleaseStringChars(env, jstr, cstr); // 发生异常后释放分配内存
         return; 
     }
```



### 2 抛出异常

ThrowNew：在当前线程触发一个异常，并自定义输出异常信息 
`jint (JNICALL *ThrowNew) (JNIEnv *env, jclass clazz, const char *msg);` 

Throw：丢弃一个现有的异常对象，在当前线程触发一个新的异常 
`jint (JNICALL *Throw) (JNIEnv *env, jthrowable obj);` 

FatalError：致命异常，用于输出一个异常信息，并终止当前VM实例（即退出程序） 

`void (JNICALL *FatalError) (JNIEnv *env, const char *msg);`



### 3 示例

```c
jclass jclass1 = (*env)->FindClass(env, "com/test/JNIController");
    if (jclass1 == 0) {
        return;
    }

    jmethodID methodID = (*env)->GetMethodID(env, jclass1, "exitProcessCallBack", "(III)V");
    if (methodID == 0) {
        return;
    }

    (*env)->CallVoidMethod(env, local_object, methodID, code, uploaded, all);

    if ((*env)->ExceptionCheck){
        (*env)->ExceptionDescribe(env);     // 打印异常的堆栈信息 
        (*env)->ExceptionClear(env);        // 清除异常堆栈信息 
        (*env)->ThrowNew(env,(*env)->FindClass(env,"java/lang/Exception"),"JNI出现异常！");
    }
```



### 4 总结

1. 当调用一个JNI函数后，必须先检查、处理、清除异常后再做其它 JNI 函数调用，否则会产生不可预知的结果。 
2. 一旦发生异常，立即返回，让调用者处理这个异常。或 调用 ExceptionClear 清除异常，然后执行自己的异常处理代码。 