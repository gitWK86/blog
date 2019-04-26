---
title: SDL2 PCM音频播放
date: 2019-04-19 09:47:03
categories: 
- SDL2
tags:
- 音视频
- SDL2
---

SDL2文章列表

[SDL2入门](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)

[SDL2事件处理](https://david1840.github.io/2019/04/15/SDL2%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/)

[SDL2纹理渲染](https://david1840.github.io/2019/04/16/SDL2%E7%BA%B9%E7%90%86%E6%B8%B2%E6%9F%93/)

本来计划写FFmpeg+SDL2视频播放，但是发现要说的内容有点多，所以还是先从简单的音频数据播放开始，一步一步来。

### 打开音频设备

```c
int SDLCALL SDL_OpenAudio(SDL_AudioSpec * desired,
                          SDL_AudioSpec * obtained);
// desired：期望的参数。
// obtained：实际音频设备的参数，一般情况下设置为NULL即可。
```

#### SDL_AudioSpec

```c
//在这个结构体中包含了音频的各种参数
typedef struct SDL_AudioSpec
{
    int freq;                   /**< 音频采样率*/
    SDL_AudioFormat format;     /**< 音频数据格式 */
    Uint8 channels;             /**< 声道数: 1 单声道, 2 立体声 */
    Uint8 silence;              /**< 设置静音的值*/
    Uint16 samples;             /**< 音频缓冲区中的采样个数，要求必须是2的n次*/
    Uint16 padding;             /**< 考虑到兼容性的一个参数*/
    Uint32 size;                /**< 音频缓冲区的大小，以字节为单位*/
    SDL_AudioCallback callback; /**< 填充音频缓冲区的回调函数 */
    void *userdata;             /**< 用户自定义的数据 */
} SDL_AudioSpec;
```

#### SDL_AudioCallback

当音频设备需要更多数据的时候会调用该回调函数。

```
// userdata：SDL_AudioSpec结构中的用户自定义数据，一般情况下可以不用。
// stream：该指针指向需要填充的音频缓冲区。
// len：音频缓冲区的大小（以字节为单位）。
void (SDLCALL * SDL_AudioCallback) (void *userdata,
                                    Uint8 *stream,
                                    int len);
                                         
```

### 播放音频数据

```c
// 当pause_on设置为0的时候即可开始播放音频数据。设置为1的时候，将会播放静音的值。
void SDLCALL SDL_PauseAudio(int pause_on)
```



### 播放PCM音频Demo

```c
#include <stdio.h>
#include <tchar.h>
#include <SDL_types.h>
#include "SDL.h"

static Uint8 *audio_chunk;
static Uint32 audio_len;
static Uint8 *audio_pos;
int pcm_buffer_size = 4096;

//回调函数，音频设备需要更多数据的时候会调用该回调函数
void read_audio_data(void *udata, Uint8 *stream, int len) {
    SDL_memset(stream, 0, len);
    if (audio_len == 0)
        return;
    len = (len > audio_len ? audio_len : len);

    SDL_MixAudio(stream, audio_pos, len, SDL_MIX_MAXVOLUME);
    audio_pos += len;
    audio_len -= len;
}

int WinMain(int argc, char *argv[]) {
    
    if (SDL_Init(SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
        printf("Could not initialize SDL - %s\n", SDL_GetError());
        return -1;
    }
  
    SDL_AudioSpec spec;
    spec.freq = 44100;//根据你录制的PCM采样率决定
    spec.format = AUDIO_S16SYS;
    spec.channels = 1; //单声道
    spec.silence = 0;
    spec.samples = 1024;
    spec.callback = read_audio_data;
    spec.userdata = NULL;

    if (SDL_OpenAudio(&spec, NULL) < 0) {
        printf("can't open audio.\n");
        return -1;
    }

    FILE *fp = fopen("C:\\Users\\lenovo\\Desktop\\1111111.pcm", "rb+");
    if (fp == NULL) {
        printf("cannot open this file\n");
        return -1;
    }
    char *pcm_buffer = (char *) malloc(pcm_buffer_size);

    //播放
    SDL_PauseAudio(0);

    while (1) {
        if (fread(pcm_buffer, 1, pcm_buffer_size, fp) != pcm_buffer_size) { //从文件中读取数据，剩下的就交给音频设备去完成了，它播放完一段数据后会执行回调函数，获取等多的数据
            break;
        }

        audio_chunk = (Uint8 *) pcm_buffer;
        audio_len = pcm_buffer_size; //长度为读出数据长度，在read_audio_data中做减法
        audio_pos = audio_chunk;

        while (audio_len > 0) //判断是否播放完毕
            SDL_Delay(1);
    }
    free(pcm_buffer);
    SDL_Quit();

    return 0;
}
```

OK！第一步完成，能正常播放出声音了。

想要录制PCM自己试一下？可以试试用这个[Android音视频(五) OpenSL ES录制、播放音频(带源码)](https://david1840.github.io/2019/02/11/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%BA%94-OpenSL-ES%E6%92%AD%E6%94%BE%E9%9F%B3%E9%A2%91/)录制一段。

[源码 GitHub-SimplePlayer-pcm_player](https://github.com/David1840/SimplePlayer/blob/master/pcm_player.c)



====== 更新 完成了FFmpeg+SDL2播放音频流的开发，代码请看[源码 GitHub-SimplePlayer-audio_player](https://github.com/David1840/SimplePlayer/blob/master/audio_player.c)