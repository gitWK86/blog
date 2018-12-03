---
title: Android JNI学习-使用第三方SO库
date: 2018-12-03 19:09:27
categories: 
- Android开发
tags:
- JNI
- NDK
---

正在准备中的项目里，有一部分打算直接移殖Linux开发组在之前就完成的功能，他们是使用C语言开发。考虑到维护的问题，准备让他们将代码打包成so文件，再引用到我的项目中。这样也就相当于我去引用一个第三方库，并且这个库中的代码格式也不一定是我们JNI开发时规定的命名，因此，需要通过我自己的C文件再去调用so库中的方法。

## 生成SO库

### Native方法

新建项目JNISODemo，在MainActivity中定义Native方法：

```
public native String getString();
```

### 头文件生成

.h文件的生成是在命令行cd到main目录下，再使用javah生成。

这次是想介绍下快捷方式。

**File -> Settings -> Tools -> External tools -> +**



![](/Android-JNI学习-使用第三方SO库/jni-so1.png)



![](/Android-JNI学习-使用第三方SO库/jni-so2.png)



在我们声明native方法的类上点击右键，javah，输入命名（我命名为Test.h)，之后就会先自动创建一个jni文件夹，然后生成一个Test.h文件，copy Test.h，并将命名改为Test.c。

### 完成C代码

在Test.c中简单完成下我们定义的方法，返回一个字符串：



```C
#include <jni.h>
/* Header for class com_david_jnisodemo_MainActivity */

#ifndef _Included_com_david_jnisodemo_MainActivity
#define _Included_com_david_jnisodemo_MainActivity
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_david_jnisodemo_MainActivity
 * Method:    getString
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_david_jnisodemo_MainActivity_getString
        (JNIEnv *env, jobject instance) {
    return (*env)->NewStringUTF(env, "This is a test!");
}

#ifdef __cplusplus
}
#endif
#endif
```



### CMakeList.txt

```make
cmake_minimum_required(VERSION 3.4.1)


add_library( # Sets the name of the library.
             Test
             
             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/jni/Test.c )


find_library(log-lib log )

target_link_libraries( # Specifies the target library.
                       Test
                       ${log-lib} )
```



### build.gradle

```
ndk {
    ldLibs "log"//实现__android_log_print
    abiFilters  "armeabi-v7a" //平台配置，因为在Android上，就只写了一个
}
```



### 编译

编译完成后就如下图，产生一个libTest.so的文件，这就是我们要的。把它当做Linux最后打包成的so文件。

![](/Android-JNI学习-使用第三方SO库/jni-so3.png)





## 导入第三方so

新建一个项目JNIUseSoDemo，项目结构如下。同样是在MainActivity中定义Native方法，生成UseSo.c。将我们上一步生成的so文件拷贝到jniLibs下（armeabi-v7a是平台）。以及上一步中的头文件也拷贝到jni下。

![](/Android-JNI学习-使用第三方SO库/jni-so4.png)



### 完成C代码

我在UseSo中getString方法去调用了so库中的getString方法。

```C
#include <jni.h>
#include "Test.h" //so库的头文件，必须要引用！

/* Header for class com_david_jniusesodemo_MainActivity */

#ifndef _Included_com_david_jniusesodemo_MainActivity
#define _Included_com_david_jniusesodemo_MainActivity
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_david_jniusesodemo_MainActivity
 * Method:    getString
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_david_jniusesodemo_MainActivity_getString
        (JNIEnv *env, jobject instance) {
    
    return Java_com_david_jnisodemo_MainActivity_getString(env, instance);
}

#ifdef __cplusplus
}
#endif
#endif
```

当然还没有完成。



### CMakeList.txt

在CMake中将LibTest.so导入工程



```
cmake_minimum_required(VERSION 3.4.1)

add_library( # Sets the name of the library.
             UseSo

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/jni/UseSo.c )
             
#导入第三方so包，并声明为 IMPORTED 属性，指明只是想把 so 导入到项目中
add_library( Test
             SHARED
             IMPORTED )
             
#指明 so 库的路径，CMAKE_SOURCE_DIR 表示 CMakeLists.txt 的路径             
set_target_properties(
             Test
             PROPERTIES IMPORTED_LOCATION
             ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libTest.so )

#指明头文件路径，不然会提示找不到 so 的方法
include_directories(src/main/jni/)

find_library(log-lib

              log )

target_link_libraries( # Specifies the target library.
                       UseSo

                       Test

                       ${log-lib} )
```



### build.gradle



```
ndk {
    abiFilters 'armeabi-v7a'
}
```



### 最终调用



```Java
public class MainActivity extends Activity {

    static {
        System.loadLibrary("UseSo"); //加载SO库
    }

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Log.e("TEST",getString());
    }


    public native String getString(); //它会调用Java_com_david_jniusesodemo_MainActivity_getString方法，然后该方法又回去调用so库中的Java_com_david_jnisodemo_MainActivity_getString方法，得到返回字符串。

}
```

![](/Android-JNI学习-使用第三方SO库/jni-so5.png)



验证没有问题！