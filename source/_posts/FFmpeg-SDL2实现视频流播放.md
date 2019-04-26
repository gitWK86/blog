---
title: FFmpeg+SDL2实现视频流播放
date: 2019-04-22 15:47:51
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

本篇博客使用FFmpeg+SDL2完成播放视频流Demo（仅播放视频），所有相关知识在之前的博客中都有提到，稍作整理完成。

### 流程图：

FFmpeg解码视频流：

![](/FFmpeg-SDL2实现视频流播放/FFmpeg.png)

SDL2显示YUV数据：

![](/FFmpeg-SDL2实现视频流播放/SDL2.png)

### 

### 源码

```c
#include <stdio.h>
#include <SDL.h>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>

int WinMain(int argc, char *argv[]) {
    int ret = -1;
    char *file = "C:\\Users\\lenovo\\Desktop\\fengjing.mp4";
    
    AVFormatContext *pFormatCtx = NULL; 
    int i, videoStream;
    AVCodecParameters *pCodecParameters = NULL; 
    AVCodecContext *pCodecCtx = NULL;
    AVCodec *pCodec = NULL;
    AVFrame *pFrame = NULL;
    AVPacket packet;

    SDL_Rect rect;
    Uint32 pixformat;
    SDL_Window *win = NULL;
    SDL_Renderer *renderer = NULL;
    SDL_Texture *texture = NULL;
    
    //默认窗口大小
    int w_width = 640;
    int w_height = 480;
    
    //SDL初始化
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Could not initialize SDL - %s\n", SDL_GetError());
        return ret;
    }


    // 打开输入文件
    if (avformat_open_input(&pFormatCtx, file, NULL, NULL) != 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Couldn't open  video file!");
        goto __FAIL; 
    }

    //找到视频流
    videoStream = av_find_best_stream(pFormatCtx, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);
    if (videoStream == -1) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Din't find a video stream!");
        goto __FAIL;// Didn't find a video stream
    }

    // 流参数
    pCodecParameters = pFormatCtx->streams[videoStream]->codecpar;

    //获取解码器
    pCodec = avcodec_find_decoder(pCodecParameters->codec_id);
    if (pCodec == NULL) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Unsupported codec!\n");
        goto __FAIL; // Codec not found
    }

    // 初始化一个编解码上下文
    pCodecCtx = avcodec_alloc_context3(pCodec);
    if (avcodec_parameters_to_context(pCodecCtx, pCodecParameters) != 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Couldn't copy codec context");
        goto __FAIL;// Error copying codec context
    }

    // 打开解码器
    if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to open decoder!\n");
        goto __FAIL; // Could not open codec
    }

    // Allocate video frame
    pFrame = av_frame_alloc();

    w_width = pCodecCtx->width;
    w_height = pCodecCtx->height;

    //创建窗口
    win = SDL_CreateWindow("Media Player",
                           SDL_WINDOWPOS_UNDEFINED,
                           SDL_WINDOWPOS_UNDEFINED,
                           w_width, w_height,
                           SDL_WINDOW_OPENGL | SDL_WINDOW_RESIZABLE);
    if (!win) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to create window by SDL");
        goto __FAIL;
    }

    //创建渲染器
    renderer = SDL_CreateRenderer(win, -1, 0);
    if (!renderer) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to create Renderer by SDL");
        goto __FAIL;
    }

    pixformat = SDL_PIXELFORMAT_IYUV;//YUV格式
    // 创建纹理
    texture = SDL_CreateTexture(renderer,
                                pixformat,
                                SDL_TEXTUREACCESS_STREAMING,
                                w_width,
                                w_height);


    //读取数据
    while (av_read_frame(pFormatCtx, &packet) >= 0) {
        if (packet.stream_index == videoStream) {
            //解码
            avcodec_send_packet(pCodecCtx, &packet);
            while (avcodec_receive_frame(pCodecCtx, pFrame) == 0) {

                SDL_UpdateYUVTexture(texture, NULL,
                                     pFrame->data[0], pFrame->linesize[0],
                                     pFrame->data[1], pFrame->linesize[1],
                                     pFrame->data[2], pFrame->linesize[2]);

                // Set Size of Window
                rect.x = 0;
                rect.y = 0;
                rect.w = pCodecCtx->width;
                rect.h = pCodecCtx->height;

                //展示
                SDL_RenderClear(renderer);
                SDL_RenderCopy(renderer, texture, NULL, &rect);
                SDL_RenderPresent(renderer);
            }
        }

        av_packet_unref(&packet);

        // 事件处理
        SDL_Event event;
        SDL_PollEvent(&event);
        switch (event.type) {
            case SDL_QUIT:
                goto __QUIT;
            default:
                break;
        }


    }

    __QUIT:
    ret = 0;

    __FAIL:
    // Free the YUV frame
    if (pFrame) {
        av_frame_free(&pFrame);
    }

    // Close the codec
    if (pCodecCtx) {
        avcodec_close(pCodecCtx);
    }

    if (pCodecParameters) {
        avcodec_parameters_free(&pCodecParameters);
    }

    // Close the video file
    if (pFormatCtx) {
        avformat_close_input(&pFormatCtx);
    }

    if (win) {
        SDL_DestroyWindow(win);
    }

    if (renderer) {
        SDL_DestroyRenderer(renderer);
    }

    if (texture) {
        SDL_DestroyTexture(texture);
    }

    SDL_Quit();

    return ret;
}

```

![](/FFmpeg-SDL2实现视频流播放/1.png)

这个Demo目前只是通过一个while循环将视频播放出来，所以可以播放视频但是速度不正常，并且没有声音，这些问题会在后面一一解决，最后完成一个简易的播放器。

[源码 GitHub-SimplePlayer](https://github.com/David1840/SimplePlayer)



====== 更新 2019-04-25，线程操作，使画面显示40ms一帧 ======

```c
#define REFRESH_EVENT  (SDL_USEREVENT + 1) //刷新事件
#define BREAK_EVENT  (SDL_USEREVENT + 2) // 退出事件

int thread_exit = 0;
int thread_pause = 0;

//线程每40ms刷新一次
int video_refresh_thread(void *data) {
    thread_exit = 0;
    thread_pause = 0;

    while (!thread_exit) {
        if (!thread_pause) {
            SDL_Event event;
            event.type = REFRESH_EVENT;
            SDL_PushEvent(&event);// 发送刷新事件
        }
        SDL_Delay(40);
    }
    thread_exit = 0;
    thread_pause = 0;
    //Break
    SDL_Event event;
    event.type = BREAK_EVENT;
    SDL_PushEvent(&event);

    return 0;
}


```

```c
//创建线程
SDL_CreateThread(video_refresh_thread, "Video Thread", NULL);

for (;;) {
    SDL_WaitEvent(&event);//使用时间驱动，每40ms执行一次
    if (event.type == REFRESH_EVENT) {
        while (1) {
            if (av_read_frame(pFormatCtx, &packet) < 0) 
                thread_exit = 1;
           
            if (packet.stream_index == videoStream)
                break;
        }

        if (packet.stream_index == videoStream) {
            avcodec_send_packet(pCodecCtx, &packet);
            while (avcodec_receive_frame(pCodecCtx, pFrame) == 0) {

                SDL_UpdateYUVTexture(texture, NULL,
                                     pFrame->data[0], pFrame->linesize[0],
                                     pFrame->data[1], pFrame->linesize[1],
                                     pFrame->data[2], pFrame->linesize[2]);

                // Set Size of Window
                rect.x = 0;
                rect.y = 0;
                rect.w = pCodecCtx->width;
                rect.h = pCodecCtx->height;

                SDL_RenderClear(renderer);
                SDL_RenderCopy(renderer, texture, NULL, &rect);
                SDL_RenderPresent(renderer);
            }
            av_packet_unref(&packet);
        }
    } else if (event.type == SDL_KEYDOWN) {
        if (event.key.keysym.sym == SDLK_SPACE) { //空格键暂停
            thread_pause = !thread_pause;
        }
        if (event.key.keysym.sym== SDLK_ESCAPE){ // ESC键退出
            thread_exit = 1;
        }
    } else if (event.type == SDL_QUIT) {
        thread_exit = 1;
    } else if (event.type == BREAK_EVENT) {
        break;
    }
}
```

