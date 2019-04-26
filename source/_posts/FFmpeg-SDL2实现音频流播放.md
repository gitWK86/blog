---
title: FFmpeg+SDL2实现音频流播放
date: 2019-04-26 14:54:38
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

[SDL2 PCM音频播放](https://david1840.github.io/2019/04/19/SDL2%E9%9F%B3%E9%A2%91%E6%92%AD%E6%94%BE/)

[FFmpeg+SDL2实现视频流播放](https://david1840.github.io/2019/04/22/FFmpeg-SDL2%E5%AE%9E%E7%8E%B0%E8%A7%86%E9%A2%91%E6%B5%81%E6%92%AD%E6%94%BE/)

之前完成了PCM音频的播放，这次实现的是FFmpeg+SDL2播放任意视频中的音频流。

整体的流程和视频流播放类似，需要了解下的就是 **SwrContext 重采样结构体**

重采样结构体,就是改变音频的采样率、sample format、声道数等参数，使之按照我们期望的参数输出,当然是原有的音频参数不满足我们的需求，比如在FFMPEG解码音频的时候，不同的音源有不同的格式，采样率等，在解码后的数据中的这些参数也会不一致，如果我们接下来需要使用解码后的音频数据做其他操作，而这些参数的不一致导致会有很多额外工作，此时直接对其进行重采样，获取我们制定的音频参数，这样就会方便很多。

通过重采样，我们可以对 sample rate(采样率)、sample format(采样格式)、channel layout(通道布局，可以通过此参数获取声道数)进行调节。

### SwrContext常用函数

#### swr_alloc

```c
// 用于申请一个SwrContext结构体
struct SwrContext *swr_alloc(void);
```

#### swr_init

```c
// 当设置好相关的参数后，使用此函数来初始化SwrContext结构体
int swr_init(struct SwrContext *s);

```

#### swr_alloc_set_opts

```c
//分配SwrContext并设置/重置常用的参数。参数包含了输入输出参数中sample rate(采样率)、sample format(采样格式)、channel layout等参数
函数原型：struct SwrContext *swr_alloc_set_opts(struct SwrContext *s,
                                      int64_t out_ch_layout,
                                      enum AVSampleFormat out_sample_fmt, 
                                      int out_sample_rate,
                                      int64_t  in_ch_layout, 
                                      enum AVSampleFormat  in_sample_fmt, 
                                      int  in_sample_rate,
                                      int log_offset, 
                                      void *log_ctx);

```

#### swr_convert

```c
// 将输入的音频按照定义的参数进行转换，并输出
int swr_convert(struct SwrContext *s, uint8_t **out, int out_count,
                                const uint8_t **in , int in_count);
```

#### swr_free

```c
// 释放掉SwrContext结构体并将此结构体置为NULL;
void swr_free(struct SwrContext **s);
```



### 示例代码

```c
//
// Created by 刘伟 on 2019/4/26.
//

#include <stdio.h>
#include <SDL_types.h>
#include "SDL.h"

#include "libswresample/swresample.h"
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>

static Uint8 *audio_chunk;
static Uint32 audio_len;
static Uint8 *audio_pos;

#define MAX_AUDIO_FRAME_SIZE 19200


//音频设备需要更多数据的时候会调用该回调函数
void read_audio_data(void *udata, Uint8 *stream, int len) {

    //首先使用SDL_memset()将stream中的数据设置为0
    SDL_memset(stream, 0, len);
    if (audio_len == 0)
        return;
    len = (len > audio_len ? audio_len : len);

    SDL_MixAudio(stream, audio_pos, len, SDL_MIX_MAXVOLUME);
    audio_pos += len;
    audio_len -= len;
}

int WinMain(int argc, char *argv[]) {
    char *file = "C:\\Users\\lenovo\\Desktop\\1080p.mov";

    AVFormatContext *pFormatCtx = NULL; 

    int i, audioStream = -1;

    AVCodecParameters *pCodecParameters = NULL;
    AVCodecContext *pCodecCtx = NULL;

    AVCodec *pCodec = NULL; 
    AVFrame *pFrame = NULL;
    AVPacket *packet;
    uint8_t *out_buffer;

    int64_t in_channel_layout;
    
    struct SwrContext *au_convert_ctx;

    if (avformat_open_input(&pFormatCtx, file, NULL, NULL) != 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to open video file!");
        return -1; // Couldn't open file
    }

    audioStream = av_find_best_stream(pFormatCtx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);

    if (audioStream == -1) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Din't find a video stream!");
        return -1;// Didn't find a video stream
    }

    // 音频流参数
    pCodecParameters = pFormatCtx->streams[audioStream]->codecpar;

    // 获取解码器
    pCodec = avcodec_find_decoder(pCodecParameters->codec_id);
    if (pCodec == NULL) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Unsupported codec!\n");
        return -1; // Codec not found
    }

    // Copy context
    pCodecCtx = avcodec_alloc_context3(pCodec);
    if (avcodec_parameters_to_context(pCodecCtx, pCodecParameters) != 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Couldn't copy codec context");
        return -1;// Error copying codec context
    }

    // Open codec
    if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to open decoder!\n");
        return -1; // Could not open codec
    }
    packet = (AVPacket *) av_malloc(sizeof(AVPacket));
    av_init_packet(packet);
    pFrame = av_frame_alloc();

    uint64_t out_channel_layout = AV_CH_LAYOUT_STEREO;//输出声道
    int out_nb_samples = 1024;
    enum AVSampleFormat out_sample_fmt = AV_SAMPLE_FMT_S16;//输出格式S16
    int out_sample_rate = 44100;
    int out_channels = av_get_channel_layout_nb_channels(out_channel_layout);

    int out_buffer_size = av_samples_get_buffer_size(NULL, out_channels, out_nb_samples, out_sample_fmt, 1);
    out_buffer = (uint8_t *) av_malloc(MAX_AUDIO_FRAME_SIZE * 2);


    //Init
    if (SDL_Init(SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
        printf("Could not initialize SDL - %s\n", SDL_GetError());
        return -1;
    }

    SDL_AudioSpec spec;
    spec.freq = out_sample_rate;
    spec.format = AUDIO_S16SYS;
    spec.channels = out_channels;
    spec.silence = 0;
    spec.samples = out_nb_samples;
    spec.callback = read_audio_data;
    spec.userdata = pCodecCtx;

    if (SDL_OpenAudio(&spec, NULL) < 0) {
        printf("can't open audio.\n");
        return -1;
    }

    in_channel_layout = av_get_default_channel_layout(pCodecCtx->channels);
    printf("in_channel_layout --->%d\n", in_channel_layout);
    au_convert_ctx = swr_alloc();
    au_convert_ctx = swr_alloc_set_opts(au_convert_ctx, out_channel_layout, out_sample_fmt, out_sample_rate,in_channel_layout, pCodecCtx->sample_fmt, pCodecCtx->sample_rate, 0, NULL);
    swr_init(au_convert_ctx);

    SDL_PauseAudio(0);

    while (av_read_frame(pFormatCtx, packet) >= 0) {
        if (packet->stream_index == audioStream) {
            avcodec_send_packet(pCodecCtx, packet);
            while (avcodec_receive_frame(pCodecCtx, pFrame) == 0) {
                swr_convert(au_convert_ctx, &out_buffer, MAX_AUDIO_FRAME_SIZE, (const uint8_t **) pFrame->data,pFrame->nb_samples); // 转换音频
            }

            audio_chunk = (Uint8 *) out_buffer;
            audio_len = out_buffer_size;
            audio_pos = audio_chunk;

            while (audio_len > 0) {
                SDL_Delay(1);//延迟播放
            }
        }
        av_packet_unref(packet);
    }
    swr_free(&au_convert_ctx);
    SDL_Quit();

    return 0;
}
```



代码请查看[源码 GitHub-SimplePlayer-audio_player](https://github.com/David1840/SimplePlayer/blob/master/audio_player.c)