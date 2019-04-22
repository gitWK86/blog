---
title: SDL2纹理渲染
date: 2019-04-16 10:43:16
categories: 
- SDL2
tags:
- 音视频
- SDL2
---

SDL2第三篇。

[SDL2入门](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)

[SDL2事件处理](https://david1840.github.io/2019/04/15/SDL2%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/)

接下来就看下如何使用SDL如何通过SDL_Texture在窗口绘制图像。

先了解几个纹理渲染相关API：

#### 创建纹理

```c
SDL_Texture* SDL_CreateTexture(SDL_Renderer * renderer, 
						Uint32 format, 
                        int access,
                        int w, int h);
```

format: 像素格式，YUV或RGB

access: 指明Texture的类型。可以是 Stream(视频)，也可以是Target一般的类型。

#### 销毁纹理

```c
void SDL_DestroyTexture(SDL_Texture* texture)
```

#### 渲染目标

```c
//将渲染目标定为纹理
int SDL_SetRenderTarget(SDL_Renderer *renderer,
                        SDL_Texture *texture);
```

#### 纹理拷贝

```c
//会将纹理拷贝到显卡上去，显卡会计算出最终图形并渲染到窗口中
int SDL_RenderCopy(SDL_Renderer*   renderer,
                   SDL_Texture*    texture,
                   const SDL_Rect* srcrect,
                   const SDL_Rect* dstrect)
```

srcrect: 指定 Texture 中要渲染的一部分。如果将 Texture全部输出，可以设置它为 NULL。

dstrect: 指定输出的空间大小。

### 简单示例

在前面Demo的基础上做了一定修改，简单实现一个正方形在界面中随机显示。

```c
#include <stdio.h>
#include <SDL2/SDL.h>
int WinMain() {
    int quit = 1;
    SDL_Window *window = NULL;
    SDL_Renderer *renderer = NULL;
    SDL_Texture *sdlTexture = NULL;
    SDL_Event event;
    SDL_Rect rect; // 长方形，原点在左上角
    rect.w = 50;
    rect.h = 50;

    SDL_Init(SDL_INIT_VIDEO);//初始化函数,可以确定希望激活的子系统

    window = SDL_CreateWindow("My First Window",
                              SDL_WINDOWPOS_UNDEFINED,
                              SDL_WINDOWPOS_UNDEFINED,
                              640,
                              480,
                              SDL_WINDOW_OPENGL | SDL_WINDOW_RESIZABLE);// 创建窗口

    if (!window) {
        return -1;
    }
    renderer = SDL_CreateRenderer(window, -1, 0);//基于窗口创建渲染器
    if (!renderer) {
        return -1;
    }
//    SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255); //设置渲染器颜色
//    SDL_RenderClear(renderer); //用指定的颜色清空缓冲区
//    SDL_RenderPresent(renderer); //将缓冲区中的内容输出到目标窗口上

    sdlTexture = SDL_CreateTexture(renderer,
                                   SDL_PIXELFORMAT_RGBA8888,
                                   SDL_TEXTUREACCESS_TARGET,
                                   640,
                                   480); //创建纹理

    if (!sdlTexture) {
        return -1;
    }

    while (quit) {
        SDL_PollEvent(&event); // SDL_WaitEvent在这里就不太适合，只有在事件发生时才会触发，其余时间都是阻塞状态
        switch (event.type) {
            case SDL_QUIT:
                SDL_Log("quit");
                quit = 0;
                break;
            default:
                SDL_Log("event type:%d", event.type);
        }
        rect.x = rand() % 600;
        rect.y = rand() % 400;

        SDL_SetRenderTarget(renderer, sdlTexture); // 设置渲染目标为纹理
        SDL_SetRenderDrawColor(renderer, 0, 0, 0, 0); // 纹理背景为黑色
        SDL_RenderClear(renderer); //清屏

        SDL_RenderDrawRect(renderer, &rect); //绘制一个长方形
        SDL_SetRenderDrawColor(renderer, 255, 255, 255, 255); //长方形为白色
        SDL_RenderFillRect(renderer, &rect);

        SDL_SetRenderTarget(renderer, NULL); //恢复默认，渲染目标为窗口
        SDL_RenderCopy(renderer, sdlTexture, NULL, NULL); //拷贝纹理到CPU

        SDL_RenderPresent(renderer); //输出到目标窗口上
    }

    SDL_DestroyTexture(sdlTexture);
    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window); //销毁窗口
    SDL_Quit();
    return 0;
}
```

上面这个Demo就是最简单的纹理渲染流程。

接下来再认识两个API：

#### 更新纹理

```c
//两个API功能相同，但是SDL_UpdateYUVTexture直接将Y、U、V分量传入，可以减少CPU计算量，更快一些

int SDL_UpdateTexture(SDL_Texture * texture, //想要更新的纹理
                      const SDL_Rect * rect, //更新的像素矩形，传NULL则表示为整个纹理
                      const void *pixels, //像素数据
                      int pitch);//一行像素数据的字节数

int SDL_UpdateYUVTexture(SDL_Texture * texture,
                         const SDL_Rect * rect,
                         const Uint8 *Yplane, int Ypitch,
                         const Uint8 *Uplane, int Upitch,
                         const Uint8 *Vplane, int Vpitch);
```

这两个API将会在视频播放中发挥重要作用，下一篇博客将会结合FFmpeg实现一个视频播放器。