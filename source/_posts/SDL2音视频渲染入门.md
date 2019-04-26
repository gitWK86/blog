---
title: SDL2音视频渲染入门
date: 2019-04-11 14:23:25
categories: 
- SDL2
tags:
- 音视频
- SDL2
---

SDL， “Simple DirectMedia Layer”，它是一套开放源代码的跨平台多媒体开发库，使用C语言写成。其主要用于游戏开发中的多媒体处理，如视频渲染，音频播放，鼠标/键盘控制等操作。它对外接供了一套统一的接口，但在内部，它会根据不同平台调用不同的底层 API库。如在 Linux 系统下，它会使用 opengl 做渲染，而在 Window 下它会调用 D3D API进行渲染。

## SDL2安装

SDL官网下载：https://www.libsdl.org/download-2.0.php

下载Source Code（以后去看源代码也比较方便），然后进行编译安装：

```
configure --prefix=/usr/local
```

```
sudo make && make install
```

在/usr/local下就可以找到编译后的SDL2相关文件

## SDL2使用

运行环境Windows + CLion，代码通用，不同平台只需要更改依赖的SDL库即可

### CMakeList

```
cmake_minimum_required(VERSION 3.12)
project(SimplePlayer C)

set(CMAKE_C_STANDARD 99)
set(SOURCE_FILES main.c)

# 这里我使用的是自己机器上安装的SDL库，根据自己的安装路径替换
set(INC_DIR_SDL C:/cygwin64/usr/local/include/SDL2/)
set(LINK_DIR_SDL C:/cygwin64/usr/local/lib/)

include_directories(${INC_DIR_SDL})
link_directories(${LINK_DIR_SDL})

add_executable(SimplePlayer ${SOURCE_FILES})
target_link_libraries(
        SimplePlayer
        SDL2
        SDL2main)
```

### SDL的基本流程

1、初始化SDL
2、创建窗口
3、创建渲染器
4、清空缓冲区
5、绘制要显示的内容
6、最终将缓冲区内容渲染到window窗口上。
7、销毁渲染器
8、销毁窗口
9、退出SDL

下面是一个最简单的SDL程序，会显示一个640*480的窗口，窗口内部为红色，显示3秒后消失

```c
#include <SDL2/SDL.h>

int WinMain() {
    SDL_Window *window = NULL;
    SDL_Renderer *renderer = NULL;

    SDL_Init(SDL_INIT_VIDEO);// 初始化函数,可以确定希望激活的子系统

    window = SDL_CreateWindow("My First Window",
                              SDL_WINDOWPOS_UNDEFINED,
                              SDL_WINDOWPOS_UNDEFINED,
                              640,
                              480,
                              SDL_WINDOW_OPENGL | SDL_WINDOW_RESIZABLE);//  创建窗口

    if (!window) {
        return -1;
    }
    renderer = SDL_CreateRenderer(window, -1, 0);//基于窗口创建渲染器
    if (!renderer) {
        return -1;
    }
    SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255); //设置渲染器颜色 r、g、b、a
    SDL_RenderClear(renderer);//用指定的颜色清空缓冲区
    SDL_RenderPresent(renderer); //将缓冲区中的内容输出到目标窗口上。
    SDL_Delay(3000); // 延时3秒
    SDL_DestroyRenderer(renderer); //销毁渲染器
    SDL_DestroyWindow(window); // 销毁窗口
    SDL_Quit(); //退出SDL
    return 0;
}
```



![](SDL2音视频渲染入门/SDL2-1.png)



### SDL API简介

1. SDL_Init 初始化

   ```c
   int SDL_Init(Uint32 flags)
   ```

   ```
   flages：
   SDL_INIT_TIMER 定时器子系统
   SDL_INIT_AUDIO 音频子系统
   SDL_INIT_VIDEO 视频子系统，同时会初始化事件子系统
   SDL_INIT_EVENTS 事件子系统
   SDL_INIT_EVERYTHING 初始化所有子系统=
   ```

2. SDL_CreateWindow 创建窗口

   ```c
   SDL_Window* SDL_CreateWindow(const char *title,
                                int x, int y, int w,
                                int h, Uint32 flags);
   ```

   ```
   title：窗口标题
   x,y,w,h：窗口坐标
   flags:
    ::SDL_WINDOW_FULLSCREEN,//全屏         ::SDL_WINDOW_OPENGL,//使用OpenGL上下文
    ::SDL_WINDOW_HIDDEN, //窗口不可见       ::SDL_WINDOW_BORDERLESS, //无边框
    ::SDL_WINDOW_RESIZABLE,//窗口大小可变    ::SDL_WINDOW_MAXIMIZED, //窗口最大化
    ::SDL_WINDOW_MINIMIZED,//窗口最小化      ::SDL_WINDOW_INPUT_GRABBED,//输入捕获
   ```

   

3. SDL_CreateRenderer 创建渲染器

   ```c
   SDL_Renderer* SDL_CreateRenderer(SDL_Window* window,
                                    int index,
                                    Uint32 flags)
   ```

   ```
   window: 指明在哪个窗口里进行渲染
   index: 指定渲染驱动的索引号。一般指定为 -1.
   flags：
    SDL_RENDERER_SOFTWARE //The renderer is a software fallback 软件备份
    SDL_RENDERER_ACCELERATED //The renderer uses hardware acceleration 硬件加速
    SDL_RENDERER_PRESENTVSYNC //Present is synchronized with the refresh rate 刷新率同步
    SDL_RENDERER_TARGETTEXTURE //The renderer supports rendering to texture 支持渲染纹理
   ```

