---
title: FFmpeg+SDL2实现简易音视频同步播放器
date: 2019-05-05 21:58:20
categories: 
- SDL2
tags:
- 音视频
- FFmpeg
- SDL2
---

SDL2文章列表

[SDL2入门](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)

[SDL2事件处理](https://david1840.github.io/2019/04/15/SDL2%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/)

[SDL2纹理渲染](https://david1840.github.io/2019/04/16/SDL2%E7%BA%B9%E7%90%86%E6%B8%B2%E6%9F%93/)

[SDL2音频播放](https://david1840.github.io/2019/04/19/SDL2%E9%9F%B3%E9%A2%91%E6%92%AD%E6%94%BE/)

[FFmpeg+SDL2实现视频流播放](https://david1840.github.io/2019/04/22/FFmpeg-SDL2%E5%AE%9E%E7%8E%B0%E8%A7%86%E9%A2%91%E6%B5%81%E6%92%AD%E6%94%BE/)

[FFmpeg+SDL2实现音频流播放](https://david1840.github.io/2019/04/26/FFmpeg-SDL2%E5%AE%9E%E7%8E%B0%E9%9F%B3%E9%A2%91%E6%B5%81%E6%92%AD%E6%94%BE/)

### SDL线程操作

在文章开始，我们首先了解下SDL提供的线程相关的API。

##### 创建线程

```c
SDL_Thread* SDL_CreateThread(SDL_ThreadFunction fn,
                             const char* name,
                             void* data)
    
 // fn: 线程要运行的函数。
 // name: 线程名。
 // data: 函数参数。
    
typedef int (SDLCALL * SDL_ThreadFunction) (void *data);
```

##### 等待线程

```c
void SDL_WaitThread(SDL_Thread* thread,
                    int* status)
//等待线程结束
```

##### 创建互斥量

```c
SDL_mutex* SDL_CreateMutex(void)
```

##### 销毁互斥量

```c
void SDL_DestroyMutex(SDL_mutex* mutex)
```

##### 加锁

```c
int SDL_LockMutex(SDL_mutex* mutex)
```

##### 解锁

```c
int SDL_UnlockMutex(SDL_mutex* mutex)
```



![](FFmpeg-SDL2实现简易音视频同步播放器/SimplePlayer.png)