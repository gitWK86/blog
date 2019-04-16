---
title: SDL2纹理渲染
date: 2019-04-16 10:43:16
categories: 
- SDL2
tags:
- 音视频
- SDL2
---

SDL2第三篇。前面两篇博客简单介绍了SDL的[使用](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)和[事件处理]()，接下来就看下如何使用SDL通过SDL_Texture将视频渲染出来。

#### 创建纹理

```c
SDL_Texture* SDL_CreateTexture(SDL_Renderer * renderer, 
						Uint32 format, 
                        int access,
                        int w, int h);
```

format: 指明像素格式，可以是YUV，也可以是RGB

access: 指明Texture的类型。可以是 Stream(视频)，也可以是Target一般的类型。

#### 渲染

```c
int SDL_RenderCopy(SDL_Renderer*   renderer,
               SDL_Texture*    texture,
               const SDL_Rect* srcrect,
               const SDL_Rect* dstrect)
```

srcrect: 指定 Texture 中要渲染的一部分。如果将 Texture全部输出，可以设置它为 NULL。

dstrect: 指定输出的空间大小。

#### 销毁纹理

```
void SDL_DestroyTexture(SDL_Texture* texture)
```