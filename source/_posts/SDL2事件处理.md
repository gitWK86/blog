---
title: SDL2事件处理
date: 2019-04-15 15:51:06
categories: 
- SDL2
tags:
- 音视频
- SDL2
---

在上一篇[SDL2音视频渲染入门](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)中，我们只是展示了一个窗口，3秒钟后自动消失。如何让这个窗口像其他正常应用的窗口一样可以进行拖动、最小化、关闭等操作，这个时候就需要SDL的事件处理了。这里所指的事件处理就是我们通常所说的，键盘事件，鼠标事件，窗口事件等，SDL对这些事件都做了封装，提供了统一的API。

### SDL事件处理

在SDL中，将所有的事件都存放在一个队列中，然后通过一个循环从队列中取出数据，进行处理，所有对事件的操作，其实就是对队列的操作。

将上一篇中的代码`SDL_Delay(3000); // 延时3秒`改为：

```c
while (quit) {
    SDL_WaitEvent(&event);
    switch (event.type) {
        case SDL_QUIT://退出事件
            SDL_Log("quit");
            quit = 0;
            break;
        default:
            SDL_Log("event type:%d", event.type);
    }
}
```

实现一个点击窗口“x”号关闭窗口的功能。

#### 事件轮训方式

SDL_WaitEvent: 事件驱动方式，当列表中有事件存在才会触发处理流程，否则处于阻塞状态，释放 CPU

SDL_PollEvent: 轮训方式，定时不断从列表中取出数据处理（可能会导致CPU 100%） 

SDL_WaitEventTimeout: 与SDL_WaitEvent的区别时，当到达超时时间后，退出阻塞状态。

```
即然有 SDL_WaitEvent了，为什么还要有SDL_PollEvent呢？这主要是由于使用的场景不同。对于游戏来说，它要求事件的实时处理，就使用SDL_PollEvent； 而对于一些其它实时性不高的情况来说，则可以使用 SDL_WaitEvent了。
```

#### SDL事件类型

SDL_WindowEvent : Window窗口相关的事件。

SDL_KeyboardEvent : 键盘相关的事件。

SDL_MouseMotionEvent : 鼠标移动相关的事件。

SDL_QuitEvent : 退出事件。

SDL_UserEvent : 用户自定义事件。

具体信息可以查看[SDL Wiki](https://wiki.libsdl.org/SDL_Event)