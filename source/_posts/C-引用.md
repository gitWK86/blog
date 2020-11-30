---
title: C++ 引用
date: 2020-11-30 10:31:48
categories: 
- C/C++
tags:
- C++
---

# C++ 引用

引用变量是一个别名，也就是说，它是某个已存在变量的另一个名字。一旦把引用初始化为某个变量，就可以使用该引用名称或变量名称来指向变量。

 ### C++ 引用 vs 指针

引用很容易与指针混淆，它们之间有三个主要的不同：

- 不存在空引用。引用必须连接到一块合法的内存。
- 一旦引用被初始化为一个对象，就不能被指向到另一个对象。指针可以在任何时候指向到另一个对象。
- 引用必须在创建时被初始化。指针可以在任何时间被初始化。

### 创建引用

```c++
#include <iostream>

using namespace std;

int main() {
    int a;
    double b;


    // 声明引用变量
    int &c = a;
    double &d = b;

    a = 20;
    cout << "c = " << c << endl;

    b = 30.0;
    cout << "d = " << d << endl;
}

//输出结果
c = 20
d = 30
```

在这些声明中，& 读作**引用**。因此，第一个声明可以读作 "c 是一个初始化为 a 的整型引用"，第二个声明可以读作 "d 是一个初始化为 b 的 double 型引用"。

### 把引用作为参数（引用传递）

```c++
#include <iostream>

using namespace std;

// 函数声明
void swap(int &x, int &y);

int main() {
    // 局部变量声明
    int a = 100;
    int b = 200;

    cout << "交换前，a 的值：" << a << "\t";
    cout << "交换前，b 的值：" << b << endl;

    /* 调用函数来交换值 */
    swap(a, b);

    cout << "交换后，a 的值：" << a << "\t";
    cout << "交换后，b 的值：" << b << endl;

    return 0;
}

// 函数定义
void swap(int &x, int &y) {
    int temp;
    temp = x; /* 保存地址 x 的值 */
    x = y;    /* 把 y 赋值给 x */
    y = temp; /* 把 x 赋值给 y  */

    return;
}

//输出结果
交换前，a 的值：100	交换前，b 的值：200
交换后，a 的值：200	交换后，b 的值：100

```

被调函数的形参虽然也作为局部变量在栈中开辟了内存空间，但在栈中放的是由主调函数放进来的实参变量的地址。被调函数对形参的任何操作都被间接寻址，即通过栈中的存放的地址访问主调函数中的中的实参变量（相当于一个人有两个名字），因此形参在任意改动都直接影响到实参。

#### 值传递

```C++
#include <iostream>

using namespace std;

// 函数声明
void swap(int x, int y);

int main() {
    // 局部变量声明
    int a = 100;
    int b = 200;

    cout << "交换前，a 的值：" << a << "\t";
    cout << "交换前，b 的值：" << b << endl;

    /* 调用函数来交换值 */
    swap(a, b);

    cout << "交换后，a 的值：" << a << "\t";
    cout << "交换后，b 的值：" << b << endl;

    return 0;
}

// 函数定义
void swap(int x, int y) {
    int temp;
    temp = x; /* 保存地址 x 的值 */
    x = y;    /* 把 y 赋值给 x */
    y = temp; /* 把 x 赋值给 y  */

    return;
}

//输出结果
交换前，a 的值：100	交换前，b 的值：200
交换后，a 的值：100	交换后，b 的值：200
```

形参时实参的拷贝，改变函数形参并不影响函数外部的实参，这是最常用的一种传递方式，也是最简单的一种传递方式。只需要传递参数，返回值是return考虑的；使用值传递这种方式，调用函数不对实参进行操作，也就是说，即使形参的值发生改变，实参的值也完全不受影响。

### 把引用作为返回值

```c++
#include <iostream>
#include <ctime>

using namespace std;

double vals[] = {10.1, 12.6, 33.1, 24.1, 50.0};

double &setValues(int i) {
    return vals[i];   // 返回第 i 个元素的引用
}

// 要调用上面定义函数的主函数
int main() {

    cout << "改变前的值" << endl;
    for (int i = 0; i < 5; i++) {
        cout << "vals[" << i << "] = ";
        cout << vals[i] << endl;
    }

    setValues(1) = 20.23; // 改变第 2 个元素
    setValues(3) = 70.8;  // 改变第 4 个元素

    cout << "改变后的值" << endl;
    for (int i = 0; i < 5; i++) {
        cout << "vals[" << i << "] = ";
        cout << vals[i] << endl;
    }
    return 0;
}

//运行结果
改变前的值
vals[0] = 10.1
vals[1] = 12.6
vals[2] = 33.1
vals[3] = 24.1
vals[4] = 50
改变后的值
vals[0] = 10.1
vals[1] = 20.23
vals[2] = 33.1
vals[3] = 70.8
vals[4] = 50
```

setValues(1)和setValues(3)得到的是实际上的实参变量地址，所以setValues(1) = 20.23就是对实际的变量进行修改。

用引用作函数的返回值的最大的好处是在内存中不产生返回值的副本，会使 C++ 程序更容易阅读和维护。