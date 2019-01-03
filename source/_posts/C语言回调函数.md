---
title: C语言回调函数
date: 2018-12-23 14:48:51
categories: 
- C/C++
tags:
- C语言
---

想要理解C语言的回调函数，需要先理解什么是函数指针。

## 函数指针

函数指针是指向函数的指针变量。

通常我们说的指针变量是指向一个整型、字符型或数组等变量，而函数指针是指向函数。

函数指针可以像一般函数一样，用于调用函数、传递参数。

函数指针变量的声明：

```c
typedef int (*fun_ptr)(int,int); // 声明一个指向同样参数、返回值的函数指针类型
```

### 实例

以下实例声明了函数指针变量 p，指向函数 max：

```c
#include <stdio.h>
 
int max(int x, int y)
{
    return x > y ? x : y;
}
 
int main(void)
{
    /* p 是函数指针 */
    int (* p)(int, int) = & max; // &可以省略
    int a, b, c, d;
 
    printf("请输入三个数字:");
    scanf("%d %d %d", & a, & b, & c);
 
    /* 与直接调用函数等价，d = max(max(a, b), c) */
    d = p(p(a, b), c); 
 
    printf("最大的数字是: %d\n", d);
 
    return 0;
}
```

编译执行，输出结果如下：

```
请输入三个数字:1 2 3
最大的数字是: 3
```

## 回调函数

函数指针理解后就轮到我们的正主了。那么我们从三个问题开始：

1. 回调函数是什么

2. 回调函数该如何使用

3. 回调函数在什么时候用

#### 1、回调函数是什么

函数指针变量可以作为某个函数的参数来使用的，回调函数就是一个通过函数指针调用的函数。

简单讲：回调函数是由别人的函数执行时调用你实现的函数。

#### 2、回调函数该如何使用

示例1：

```c
#include <stdio.h>  
#include <stdlib.h>  
  
int fun1(void)  
{  
    printf("hello world.\n");  
    return 0;  
}  
  
void callback(int (*Pfun)())  
{  
    Pfun();  
}  
  
int main(void)  
{  
    callback(fun1);  
}
```

callback回调定义的函数fun1，传递给callback的是函数fun1的地址。fun1是一个不含参数返回值为整型的函数，如果fun含有参数，还想使用回调函数则可用下面的示例2。

示例2：

```c
#include <stdio.h>
#include <stdlib.h>

int fun2(char *s)
{
	printf("%s.\n", s);
	return 0;
}
void callback(int (*Pfun)(char *), char *s)
{
	Pfun(s);
}

int main(void)
{
	callback(fun2, "hello world");
	return 0;
}
```

#### 3、回调函数在什么时候用

通常，当我们想通过一个统一接口实现不同内容的时候，用回调函数来实现就非常合适。任何时候，如果你所编写的函数必须能够在不同的时刻执行不同的类型的工作或者执行只能由函数调用者定义的工作，你都可以用回调函数来实现。许多窗口系统就是使用回调函数连接多个动作，如拖拽鼠标和点击按钮来指定调用用户程序中的某个特定函数。