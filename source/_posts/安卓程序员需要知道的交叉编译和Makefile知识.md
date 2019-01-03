---
title: 安卓程序员需要知道的交叉编译和Makefile知识
date: 2019-01-05 15:54:16
categories: 
- C/C++
tags:
- 交叉编译
- Makefile
---

最近一直在和Linux C开发打交道，开发过程中会用到交叉编译和Makefile相关知识，但是对这块真的是没有了解，所以在网上搜索，找到一篇不错的博客。本文转载自该博客[写给安卓程序员的C/C++编译入门(作者：嘉伟咯)](https://www.jianshu.com/p/3ba79f1ade39)。如有侵权请联系删除。

## 为什么要学C/C++编译

很多的安卓程序员可能都会用Android Studio写一些简单的C/C++代码,然后通过jni去调用,但是对C/C++是如何编译的其实并没有什么概念.有人可能会问,为什么安卓程序员会需要了解C/C++是如何编译的呢?

我一直都认为,要成为一个真正的高级安卓应用开发工程师,安卓源码和C/C++是两座绕不过的大山.安卓源码自然不必多说,而C/C++流行了几十年,存在着许多优秀的开源项目,我们在处理一些特定的需求的时候,可能会需要使用到它们.如脚本语言Lua,计算机视觉库OpenCV,音视频编解码库ffmpeg,谷歌的gRPC,国产游戏引擎Cocos2dx...有些库提供了完整的安卓接口,有些提供了部分安卓接口,有些则没有.在做一些高级功能时,我们常常需要使用源码,通过裁剪和交叉编译,才能编译出可以在安卓上使用的so库.总之,安卓做深做精总避不开C/C++交叉编译。



## C/C++编译器

类似java编译器javac可以将java代码编译成class文件,C/C++也有gcc、g++、clang等多种编译器可以用于编译C/C++代码.这里我们用gcc来举例。

gcc原名为GNU C 语言编译器(GNU C Compiler),因为它原本只能处理C语言.但GCC很快地扩展,变得可处理C++。后来又扩展能够支持更多编程语言,如Fortran、Pascal、Objective-C、Java、Ada、Go以及各类处理器架构上的汇编语言等,所以改名GNU编译器套件(GNU Compiler Collection)。

使用gcc其实只需要一个命令就能将一个c文件编译成可运行程序了:

```shell
gcc test.c -o test
```

通过上面这条命令可以将test.c编译成可运行程序test.但是其实C/C++的编译是经过了好几个步骤的,我这边先给大家大概的讲一讲。

### C/C++的编译流程

C/C++的编译可以分为下面几个步骤:

![](/安卓程序员需要知道的交叉编译和Makefile知识/c-c++.png)

#### 预处理

相信学过C/C++的同学都知道"宏"这个东西,它在编译的时候会被展开替换成实际的代码,这个展开的步骤就是在预处理的时候进行的.当然,预处理并不仅仅只是做宏的展开,它还做了类似头文件插入、删除注释等操作.

预处理之后的产品依然还是C/C++代码,它在代码的逻辑上和输入的C/C++源代码是完全一样的.

我们来举一个简单的例子,写一个test.h文件和一个test.c文件:

```c
//test.h
#ifndef TEST_H            
#define TEST_H

#define A 1     
#define B 2        

/**
 * add 方法的声明
 */               
int add(int a, int b);

#endif
```



```c
//test.c
#include "test.h"

/**
 * add 方法定义
 */

int add(int a, int b) {
    return a + b;
}

int main(int argc,char* argv[]) {
    add(A, B);
    return 0;                 
}
```

然后可以通过下面这个gcc命令预处理test.c文件,并且把预处理结果写到test.i:

```
gcc -E test.c -o test.i
```

然后就能看到预处理之后的test.c到底长什么样子了:



```c

```

可以看到这里它把test.h的内容(add方法的声明)插入到了test.c的代码中,然后将A、B两个宏展开成了1和2,将注释去掉了,还在头部加上了一些信息.但是光看代码逻辑,和之前我们写的代码是完全一样的.

#### 汇编代码

可能大家都听过汇编语言这个东西,但是年轻一点的同学不一定真正见过.简单来说汇编语言是将机器语言符号化了的语言,是机器不能直接识别的低级语言.我们可以通过下面的命令,将预处理后的代码编译成汇编语言:

```
gcc -S test.i -o test.s
```



然后就能看到生成的test.s文件了,里面就是我们写的c语言代码翻译而成的汇编代码:

```
.file   "test.c"
        .text
        .globl  add
        .type   add, @function
add:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    -4(%rbp), %edx
        movl    -8(%rbp), %eax
        addl    %edx, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   add, .-add
        .globl  main
        .type   main, @function
main:
.LFB1:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movq    %rsi, -16(%rbp)
        movl    $2, %esi
        movl    $1, %edi
        call    add
        movl    $0, %eax
        leave
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE1:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609"
        .section        .note.GNU-stack,"",@progbits
```



#### 汇编

汇编这一步是将汇编代码编译成机器语言:

```
gcc -c test.s -o test.o
```

生成的test.o文件里面就是机器代码了,我们可以通过nm命令来列出test.o里面的符号:

```
nm test.o
```

得到的结果如下:

```
0000000000000000 T add
0000000000000014 T main
```



#### 链接

由于我们的例子代码比较简单只有一个test.h和test.h,所以只生成了一个.o文件,其实一般的程序都是由多个模块组合成的.链接这一步就是将多个模块的代码组合成一个可执行程序.我们可以用gcc命令将多个.o文件或者静态库、动态库链接成一个可执行文件:

```
gcc test.o -o test
```

得到的就是可执行文件test了,可以直接用下面命令运行

```
./test
```

当然是没有任何输出的,因为我们就没有做任何的打印

## 编译so库

在安卓中我们一般不会直接使用C/C++编译出来的可运行文件.用的更多的应该是so库.那要如何编译so库呢?

首先我们需要将test.c中的main函数去掉,因为so库中是不会带有main函数的:

```c
#include "test.h"

/**
 * add 方法定义
 */
int add(int a, int b){
        return a + b;
}
```

然后可以使用下面命令将test.c编译成test.so:

```
gcc -shared test.c -o test.so
```

其实也就是多了个-shared参数,指定编译的结果为动态链接库.

这里是直接将.c文件编译成so,当然也能像之前的例子一样先编译出.o文件再通过链接生成so文件.

当然一般编译动态链接库,我们还会带上-fPIC参数.

fPIC (Position-Independent Code)告诉编译器产生与位置无关代码,即产生的代码中没有绝对地址,全部使用相对地址.故而代码可以被加载器加载到内存的任意位置,都可以正确的执行.不加fPIC编译出来的so,是要再加载时根据加载到的位置再次重定位的.因为它里面的代码并不是位置无关代码.如果被多个应用程序共同使用,那么它们必须每个程序维护一份.so的代码副本了.因为.so被每个程序加载的位置都不同,显然这些重定位后的代码也不同,当然不能共享.



## 交叉编译

通过上面的例子,我们知道了一个C/C++程序是怎么从源代码一步步编译成可运行程序或者so库的.但是我们编译出来的程序或者so库只能在相同系统的电脑上使用.

例如我使用的电脑是Linux系统的,那它编译出来的程序也就只能在Linux上运行,不能在安卓或者Windows上运行.

当然正常情况下不会有人专门去到android系统下编译出程序来给安卓去用.一般我们都是在PC上编译出安卓可用的程序,在给到安卓去跑的.这种是在一个平台上生成另一个平台上的可执行代码的编译方式就叫做交叉编译.

交叉编译有是三个比较重要的概念要先说明一下:

- build : 当前你使用的计算机
- host : 你的目的是编译出来的程序可以在host上运行
- target : 普通程序没有这个概念。对于想编译出编译器的人来说此属性决定了新编译器编译出的程序可以运行在哪

如果我们想要交叉编译出安卓可运行的程序或者库的话就不能直接使用gcc去编译了.而需要使用Android NDK提供了的一套交叉编译工具链.

我们首先要下载Android NDK,然后配置好环境变量NDK_ROOT指向NDK的根目录.

然后可以通过下面命令安装交叉编译工具链:

```
$NDK_ROOT/build/tools/make-standalone-toolchain.sh \
    --platform=android-19 \
    --install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm \
    --toolchain=arm-linux-androideabi-4.9 \
    --stl=gnustl
```

然后我们就能在HOME/Android/目录下看到安装好的工具链了.进到HOME/Android/standalone-toolchains/android-toolchain-arm/bin/目录下我们可以看到有arm-linux-androideabi-gcc这个程序.

它就是gcc的安卓交叉编译版本.我们将之前使用gcc去编译的例子全部换成使用它去编译就能编译出运行在安卓上的程序了:

如下面命令生成的so库就能在安卓上通过jni调用了:

```
$HOME/Android/standalone-toolchains/android-toolchain-arm/bin/arm-linux-androideabi-gcc -shared -fPIC test.c -o test.so
```



### 不同CPU架构的编译方式

当然安卓也有很多不同的CPU架构,不同CPU架构的程序也是不一定兼容的,相信大家之前在使用Android Studio去编译so的时候也有看到编译出来的库有很多个版本像armeabi、armeabi-v7a、mips、x86等.

那这些不同CPU架构的程序又要如何编译了.

我们可以在$NDK_ROOT/toolchains目录下看到者几个目录:

```
arm-linux-androideabi-4.9
aarch64-linux-android-4.9
mipsel-linux-android-4.9
mips64el-linux-android-4.9
x86-4.9
x86_64-4.9
```

这就是不同CPU架构的交叉编译工具链了.还记得我们安装工具链的命令吗?



```
$NDK_ROOT/build/tools/make-standalone-toolchain.sh \
    --platform=android-19 \
    --install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm \
    --toolchain=arm-linux-androideabi-4.9 \
    --stl=gnust
```

toolchain参数就能指定使用哪个工具链,然后就能使用该工具链去编译该架构版本的程序了.

但是,我们看到这下面并没有armeabi-v7a的工具链,那armeabi-v7a的程序要如何编译呢?

其实armeabi-v7a的程序也是用arm-linux-androideabi-4.9去编译的,只不过在编译的时候可以带上-march=armv7-a:

```
arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so
```



## Makefile

我们前面的例子都是直接用gcc或着各个交叉编译的版本的gcc去编译C/C++代码的.在代码量不多的时候这么做还是可行的,但是如果软件一旦复杂一些,代码量一多,那么编译的命令就会十分的复杂,而且还需要考虑到多个模块之间的依赖关系.

Makefile就是一个帮助我们解决这些问题的工具.它的基本原理十分简单,先让我们看看它最最基本的用法:

```
target ... : prerequisites ...
	command
	...
	...
```

target可以是一个object file(目标文件)，也可以是一个执行文件，还可以是一个标签（label）。

prerequisites就是，要生成那个target所需要的文件或是目标。

command也就是make需要执行的命令。（任意的shell命令）

这是一个文件的依赖关系，也就是说，target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在 command中。说白一点就是说，prerequisites中如果有一个以上的文件比target文件要新的话，command所定义的命令就会被执行。这就是makefile的规则。也就是makefile中最核心的内容。

还是举我们的例子代码,首先创建一个文件,名字叫Makefile,然后写上:

```
test.so : test.c test.h                                                          
    arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so
clean :
	rm test.so
```

然后就可以用make命令去编译了.make命令会找到当前目录下的Makefile,然后比较目标文件文件和依赖文件的修改时间,如果依赖文件的修改时间比较晚,或者干脆就还没有目标文件.就会执行命令.

clean不是一个文件，它只不过是一个动作名字，有点像c语言中的lable一样，其冒号后什么也没有，那么，make就不会自动去找它的依赖性，也就不会自动执行其后所定义的命令。要执行其后的命令（不仅用于clean，其他lable同样适用），就要在make命令后明显得指出这个lable的名字。这样的方法非常有用，我们可以在一个makefile中定义不用的编译或是和编译无关的命令，比如程序的打包，程序的备份，等等。

这只是比较简单的用法，具体的Makefile知识请查看[跟我一起写Makefile](http://wiki.ubuntu.org.cn/%E8%B7%9F%E6%88%91%E4%B8%80%E8%B5%B7%E5%86%99Makefile:MakeFile%E4%BB%8B%E7%BB%8D).

## CMake

CMake是一种跨平台编译工具，比make更为高级，使用起来要方便得多。CMake主要是编写CMakeLists.txt文件，然后用cmake命令将CMakeLists.txt文件转化为make所需要的makefile文件，最后用make命令编译源码生成可执行程序或共享库（so(shared object)）。

```
#1.cmake verson，指定cmake版本 
cmake_minimum_required(VERSION 3.2)

#2.project name，指定项目的名称，一般和项目的文件夹名称对应
project(myPro)

#3.head file path，头文件目录
include_directories(include)

#4.添加需要链接的库文件目录
link_directories(include)

#5.source directory，源文件目录
aux_source_directory(src DIR_SRCS)

#6.set environment variable，设置环境变量，编译用到的源文件全部都要放到这里，否则编译能够通过，但是执行的时候会出现各种问题，比如"symbol lookup error xxxxx , undefined symbol"
set(TEST_MATH ${DIR_SRCS})

#7.add executable file，添加要编译的可执行文件
add_executable(${PROJECT_NAME} ${TEST_MATH})

#8.add link library，添加可执行文件所需要的库，比如我们用到了libm.so（命名规则：lib+name+.so），就添加该库的名称
target_link_libraries(${PROJECT_NAME} m)
```

