---
title: Android JNI开发--资源释放问题
date: 2019-08-13 14:19:18
categories: 
- Android开发
tags:
- JNI
- NDK
---

最近又在开发JNI相关的项目了。本来一切正常，坐等测试完毕发布版本，然而理想是美好的，现实是骨感的。测试跑过来跟我说在测异常流程（开发人员听到估计就头疼）的时候发生了闪退问题。我赶紧拿过来自己测，果然复现了，日志中显示`local reference table overflow (max=512)` 。嗯？JNI中出现了内存泄漏？可是我已经按照网上的例子把所有该释放的对象都释放了啊，怎么回事啊？

先简单回顾下网上常见的：

1. FindClass 

   ```c
   jclass ref= (env)->FindClass("java/lang/String");
   env->DeleteLocalRef(ref);   
   ```

2. NewString/ NewStringUTF/NewObject/NewByteArray

   ```c
   // 创建 jstring 和 char*
   jstring jstr = (*jniEnv)->CallObjectMethod(jniEnv, test1, test2);
   char* cstr = (char*) (*jniEnv)->GetStringUTFChars(jniEnv,jstr, 0);
   
   // 释放
   (*jniEnv)->ReleaseStringUTFChars(jniEnv, jstr, cstr);
   (*jniEnv)->DeleteLocalRef(jniEnv, jstr);
   ```

3. GetObjectField/GetObjectClass/GetObjectArrayElement

   ```c
   jclass ref = env->GetObjectClass(robj);
   env->DeleteLocalRef(ref);   
   ```

4. GetByteArrayElements

   ```c
   jbyte* array= (*env)->GetByteArrayElements(env,jarray,&isCopy);
   (*env)->ReleaseByteArrayElements(env,jarray,array,0);  
   ```

5. NewGlobalRef/DeleteGlobalRef

   ```
   jobject ref= env->NewGlobalRef(customObj);
   env->DeleteGlobalRef(customObj);  
   ```



开始了苦逼的代码检查之路，检查代码，上面提到的我都已经做了处理，然后考虑各种方式，各种测试，还是会在异常流程中出现闪退，令人绝望。

最后，忽然看到了这个：

```c
jbyteArray arr = (jbyteArray) (*env)->CallObjectMethod(env, local_object, methodID, java_slot,jbyteArray1);
```

从JNI中反射调用Java层方法，返回了一个字节数组，这个字节数组会被JVM回收吗？我一直认为这个是会被虚拟机回收的，但到了现在，什么都有可能了，所以我测试了一下，果然，这个数组在异常流程中被不断创建，并且没有得到回收，所以很快就出现了`local reference table overflow (max=512)`错误。找到问题根源了，赶紧检查代码，所有类似的接口全部进行修改。

```c
(*env)->DeleteLocalRef(env, arr);
```

然后再次送测，终于没有问题了。

因为这个，要解决只要一行代码的问题，花费了我大半个下午时间，所以在这里记录一下，提醒自己，以后记得释放所有在Native层中创建的本地对象！也希望能帮到遇到类似问题的朋友。