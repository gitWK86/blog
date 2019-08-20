---
title: Android JNI开发--JNI返回Java对象
date: 2019-08-20 10:32:37
categories: 
- Android开发
tags:
- JNI
- NDK
---

昨天同事问我一个JNI问题，想从Native代码中返回一个Java对象，但是网上找的例子运行就崩溃了。仔细一想，我好想也没做过这样的操作，赶紧学习一下。

从Native层返回一个Java对象，有两种操作

1. 传入一个创建好的Java对象，只在JNI代码中做赋值操作并返回
2. 完全在JNI代码中新建一个对象，赋值并返回

##### 创建一个Person类

```Java
public class Person {
    private String name;
    private int age;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

##### Native方法

```Java
//方法1 从Java层传入一个对象
public native Person getPerson(Person person);
//方法2 完全从Native代码中创建对象
public native Person getPerson2();
```

##### C++代码

###### 方法1

```c++
extern "C"
JNIEXPORT jobject JNICALL
Java_com_myapplication_MainActivity_getPerson(JNIEnv *env, jobject instance,
                                                      jobject person) {
    
    // 找到对象的Java类
    jclass myClass = env->FindClass("com/myapplication/Person");

    // 对应的Java属性
    jfieldID name = env->GetFieldID(myClass, "name", "Ljava/lang/String;");
    jfieldID age = env->GetFieldID(myClass, "age", "I");

    //属性赋值，person为传入的Java对象
    env->SetObjectField(person, name, env->NewStringUTF("liuwei"));
    env->SetIntField(person, age, 20);

    return person;
}
```

###### 方法2

```c++
extern "C"
JNIEXPORT jobject JNICALL
Java_com_myapplication_MainActivity_getPerson2(JNIEnv *env, jobject instance) {
   
    jclass myClass = env->FindClass("com/myapplication/Person");
    // 获取类的构造函数，记住这里是调用无参的构造函数
    jmethodID id = env->GetMethodID(myClass, "<init>", "()V");
    // 创建一个新的对象
    jobject person_ = env->NewObject(myClass, id);
    
    jfieldID name = env->GetFieldID(myClass, "name", "Ljava/lang/String;");
    jfieldID age = env->GetFieldID(myClass, "age", "I");

    env->SetObjectField(person_, name, env->NewStringUTF("liuwei"));
    env->SetIntField(person_, age, 20);

    return person_;
}
```



可以看到，方法1和方法2的代码区别就2行

```c++
     // 获取类的构造函数，记住这里是调用无参的构造函数
    jmethodID id = env->GetMethodID(myClass, "<init>", "()V");
    // 创建一个新的对象
    jobject person_ = env->NewObject(myClass, id);
```

在开发时 `env->GetMethodID(myClass, "<init>", "()V");`很可能会在写代码是标红，提示无法找到`<init>`,不需要理会，直接编译就好了。

##### 调用

```Java
TextView tv = findViewById(R.id.sample_text);

Person person = new Person();
//传入Java对象，返回的也是同一个对象
getPerson(person);

tv.setText(person.getName() // 方法1
           + " : " +
           getPerson2().getAge() // 方法2
    );
```

搞定！又学习了一个知识点。

对了，同事代码崩溃的问题就是Java层用了方法2，但是JNI代码却用了方法1，没有创建出一个对象，导致崩溃。