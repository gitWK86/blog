---
title: Android JNI学习-线程操作
date: 2019-02-14 10:30:18
categories: 
- Android开发
tags:
- JNI
- NDK
---

Android Native中支持的线程标准是 POSIX 线程。POSIX 线程也被简称为Pthreads，是一个线程的POSIX 标准，它为创建和处理线程定义了一个通用的API。

POSIX Thread 的Android实现是Bionic标准库的一部分，在编译的时候不需要链接任何其他的库，只需要包含一个头文件。

```c
#include <pthread.h>
```

### 创建线程

线程创建函数：

```c++
int pthread_create(
	pthread_t* thread, 
	pthread_attr_t const* attr, 
	void* (*start_routine)(void*), 
	void* arg);

```

- thread：指向 pthread_t 类型变量的指针，用它代表返回线程的句柄

- attr：指向 pthread_attr_t 结构的指针形式存在的新线程属性，可以通过该结构来指定新线程的一些属性，比如栈大小、调度优先级等，具体看 pthread_attr_t 结构的内容。如果没有特殊要求，可使用默认值，把该变量取值为 NULL 。

- 第三个参数是指向启动函数的函数指针，它的函数签名格式如下：

  ```c++
  void* start_routine(void* args)
  ```

  启动程序将线程参数看成 void 指针，返回 void 指针类型结果。

- 线程启动程序的参数，也就是函数的参数，如果不需要传递参数，它可以为 NULL 。

`pthread_create` 函数如果执行成功了则返回 0 ，如果返回其他错误代码。



```c++
void sayHello(void *){
    LOGE("say %s","hello");
}

JNIEXPORT jint JNICALL Java_com_david_JNIController_sayhello
        (JNIEnv *jniEnv, jobject instance) {
    pthread_t handles; // 线程句柄
    int ret = pthread_create(&handles, NULL, sayHello, NULL);
    if (ret != 0) {
        LOGE("create thread failed");
    } else {
        LOGD("create thread success");
    }
}
```

调用函数就可以在线程执行打印say hello了。

### 附着在Java虚拟机上

创建了线程后，只能做一些简单的Native操作，如果想要对Java层做一些操作就不行了，因为它没有Java虚拟机环境，这个时候为了和Java空间进行交互，就要把POSIX 线程附着在Java虚拟机上，然后就可以获得当前线程的 JNIEnv 指针了。

通过 `AttachCurrentThread` 方法可以将当前线程附着到 Java 虚拟机上，并且可以获得 JNIEnv 指针。而`AttachCurrentThread` 方法是由 JavaVM 指针调用的，可以在`JNI_OnLoad`函数中将JavaVM 保存为全局变量。

```c++
static JavaVM *jVm = NULL;
JNIEXPORT int JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    jVm = vm;
    return JNI_VERSION_1_6;
}

```

如上一个例子，我们想要在sayHello函数中调用一个Java层的函数`javaSayHello()`

```Java
private void javaSayHello() {
    Log.e(TAG,"java say hello");
}
```
```c++
void sayHello(void *){
    LOGE("say %s","hello");
     JNIEnv *env = NULL;
    // 将当前线程添加到 Java 虚拟机上
    if (jVm->AttachCurrentThread(&env, NULL) == 0) {
        ......
        env->CallVoidMethod(Obj, javaSayHello);
        // 从 Java 虚拟机上分离当前线程
        jVm->DetachCurrentThread();  
    }
    return NULL;
}
```

这样就在 Native 线程中调用 Java 相关的函数了。



### 等待线程返回结果

前面提到的方法是新线程运行后，该方法也就立即返回退出，执行完了。我们也可以通过另一个函数可以在等待线程执行完毕后，拿到线程执行完的结果之后再退出。

```c
int pthread_join(pthread_t pthread, void** ret_value);
```

- pthread 代表创建线程的句柄
- ret_value代表线程运行函数返回的结果

```c++
	pthread_t* handles = new pthread_t[10];
	
	for (int i = 0; i < 10; ++i) {
        pthread_t pthread;
        // 创建线程，
        int result = pthread_create(&handles[i], NULL, run, NULL;
        }
    }
    for (int i = 0; i < 10; ++i) {
        void *result = NULL; // 线程执行返回结果
        // 等待线程执行结束
        if (pthread_join(handles[i], &result) != 0) {
            env->ThrowNew(env, runtimeException, "Unable to join thread");
        } else {
	        LOGD("return value is %d",result);
        }
    }

```

 pthread_join 返回为 0 代表执行成功，非 0 则执行失败。



### 同步代码块

在Java中，JDK为我们提供了synchronized来处理多线程同步代码块。

```java
 synchronized (object.class) {
        // 业务处理
    }
```

本地代码中，JNI提供了两个函数来完成上面的同步：

（1）MonitorEnter：进入同步代码块

（2）MonitorExit：退出同步代码块

```c++
if(env->MonitorEnter(obj)!= JNI_OK){
    // 错误处理
}

// 同步代码块

// 出现错误释放代码块
if(env->ExceptionCheck()){
    if(env->MonitorExit(obj)!= JNI_OK);
       return;
}

if(env->MonitorExit(obj)!= JNI_OK){
    // 错误处理
}
```

可以发现在本地代码中处理同步代码块要比Java中复杂的多，所以，尽量用Java来做同步吧，把与同步相关的代码都移到Java中去。