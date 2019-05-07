---
title: FFmpeg音视频同步
date: 2019-05-01 15:44:16
categories: 
- FFmpeg
tags:
- 音视频
- FFmpeg
---

SDL2文章列表

[SDL2入门](https://david1840.github.io/2019/04/11/SDL2%E9%9F%B3%E8%A7%86%E9%A2%91%E6%B8%B2%E6%9F%93%E5%85%A5%E9%97%A8/)

[SDL2事件处理](https://david1840.github.io/2019/04/15/SDL2%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/)

[SDL2纹理渲染](https://david1840.github.io/2019/04/16/SDL2%E7%BA%B9%E7%90%86%E6%B8%B2%E6%9F%93/)

[SDL2音频播放](https://david1840.github.io/2019/04/19/SDL2%E9%9F%B3%E9%A2%91%E6%92%AD%E6%94%BE/)

[FFmpeg+SDL2实现视频流播放](https://david1840.github.io/2019/04/22/FFmpeg-SDL2%E5%AE%9E%E7%8E%B0%E8%A7%86%E9%A2%91%E6%B5%81%E6%92%AD%E6%94%BE/)

[FFmpeg+SDL2实现音频流播放](https://david1840.github.io/2019/04/26/FFmpeg-SDL2%E5%AE%9E%E7%8E%B0%E9%9F%B3%E9%A2%91%E6%B5%81%E6%92%AD%E6%94%BE/)

前两篇文章分别做了音频和视频的播放，要实现一个完整的简易播放器就必须要做到音视频同步播放了，而音视频同步在音视频开发中又是非常重要的知识点，所以在这里记录下音视频同步相关知识的理解。

### 音视频同步简介

从前面的学习可以知道，在一个视频文件中，音频和视频都是单独以一条流的形式存在，互不干扰。那么在播放时根据视频的帧率（Frame Rate）和音频的采样率（Sample Rate）通过简单的计算得到其在某一Frame（Sample）的播放时间分别播放，**<u>理论</u>**上应该是同步的。但是由于机器运行速度，解码效率等等因素影响，很有可能出现音频和视频不同步，例如出现视频中人在说话，却只能看到人物嘴动却没有声音，非常影响用户观看体验。

如何做到音视频同步？要知道音视频同步是一个动态的过程，同步是暂时的，不同步才是常态，需要一种随着时间会线性增长的量，视频和音频的播放速度都以该量为标准，播放快了就减慢播放速度；播放慢了就加快播放的速度，在你追我赶中达到同步的状态。目前主要有三种方式实现同步：

- **将视频和音频同步外部的时钟上**，选择一个外部时钟为基准，视频和音频的播放速度都以该时钟为标准。
- **将音频同步到视频上**，就是以视频的播放速度为基准来同步音频。
- **将视频同步到音频上**，就是以音频的播放速度为基准来同步视频。

比较主流的是第三种，将视频同步到音频上。至于为什么不使用前两种，因为一般来说，人对于声音的敏感度更高，如果频繁地去调整音频会产生杂音让人感觉到刺耳不舒服，而人对图像的敏感度就低很多了，所以一般都会采用第三种方式。

### 复习DTS、PTS和时间基

- PTS: Presentation  Time Stamp，显示渲染用的时间戳，告诉我们什么时候需要显示
- DTS: Decode Time Stamp，视频解码时的时间戳，告诉我们什么时候需要解码

在音频中PTS和DTS一般相同。但是在视频中，由于B帧的存在，PTS和DTS可能会不同。

> 实际帧顺序：I B B P
>
> 存放帧顺序：I P B B
>
> 解码时间戳：1 4 2 3
>
> 展示时间戳：1 2 3 4

- 时间基


```c
/**
 * This is the fundamental unit of time (in seconds) in terms
 * of which frame timestamps are represented.
 * 这是表示帧时间戳的基本时间单位(以秒为单位)。
**/
typedef struct AVRational{
    int num; ///< Numerator 分子
    int den; ///< Denominator 分母
} AVRational;
```

时间基是一个分数，以秒为单位，比如1/50秒，那它到底表示的是什么意思呢？以帧率为例，如果它的时间基是1/50秒，那么就表示每隔1/50秒显示一帧数据，也就是每1秒显示50帧，帧率为50FPS。

每一帧数据都有对应的PTS，在播放视频或音频的时候我们需要将PTS时间戳转化为以秒为单位的时间，用来最后的展示。那如何计算一桢在整个视频中的时间位置？

```c
static inline double av_q2d(AVRational a){
    return a.num / (double) a.den;
}

//计算一桢在整个视频中的时间位置
timestamp(秒) = pts * av_q2d(st->time_base);
```

### Audio_Clock

Audio_Clock，也就是Audio的播放时长，从开始到当前的时间。获取Audio_Clock：

```c
if (pkt->pts != AV_NOPTS_VALUE) {
    state->audio_clock = av_q2d(state->audio_st->time_base) * pkt->pts;
}
```

还没有结束，由于一个packet中可以包含多个Frame帧，packet中的PTS比真正的播放的PTS可能会早很多，可以根据Sample Rate 和 Sample Format来计算出该packet中的数据可以播放的时长，再次更新Audio_Clock。

```c
// 每秒钟音频播放的字节数 采样率 * 通道数 * 采样位数 (一个sample占用的字节数)
n = 2 * state->audio_ctx->channels;
state->audio_clock += (double) data_size /
                   (double) (n * state->audio_ctx->sample_rate);
```

 最后还有一步，在我们获取这个Audio_Clock时，很有可能音频缓冲区还有没有播放结束的数据，也就是有一部分数据实际还没有播放，所以就要在Audio_Clock上减去这部分数据的播放时间，才是真正的Audio_Clock。

```c
double get_audio_clock(VideoState *state) {
    double pts;
    int buf_size, bytes_per_sec;

    //上一步获取的PTS
    pts = state->audio_clock;
    // 音频缓冲区还没有播放的数据
    buf_size = state->audio_buf_size - state->audio_buf_index; 
    // 每秒钟音频播放的字节数
    bytes_per_sec = state->audio_ctx->sample_rate * state->audio_ctx->channels * 2;
    pts -= (double) buf_size / bytes_per_sec;
    return pts;
}
```

`get_audio_clock`中返回的才是我们最终需要的Audio_Clock，当前的音频的播放时长。

### Video_Clock

Video_Clock，视频播放到当前帧时的已播放的时间长度。

```c
avcodec_send_packet(state->video_ctx, packet);
while (avcodec_receive_frame(state->video_ctx, pFrame) == 0) {
    if ((pts = pFrame->best_effort_timestamp) != AV_NOPTS_VALUE) {
    } else {
        pts = 0;
    }
    pts *= av_q2d(state->video_st->time_base); // 时间基换算，单位为秒

    pts = synchronize_video(state, pFrame, pts);
    
    av_packet_unref(packet);
}
```

旧版的FFmpeg使用`av_frame_get_best_effort_timestamp`函数获取视频的最合适PTS，新版本的则在解码时生成了`best_effort_timestamp`。但是依然可能会获取不到正确的PTS，所以在`synchronize_video`中进行处理。

```c
double synchronize_video(VideoState *state, AVFrame *src_frame, double pts) {

    double frame_delay;

    if (pts != 0) {
        state->video_clock = pts;
    } else {
        pts = state->video_clock;// PTS错误，使用上一次的PTS值
    }
    //根据时间基，计算每一帧的间隔时间
    frame_delay = av_q2d(state->video_ctx->time_base);
    //解码后的帧要延时的时间
    frame_delay += src_frame->repeat_pict * (frame_delay * 0.5);
    state->video_clock += frame_delay;//得到video_clock,实际上也是预测的下一帧视频的时间
    return pts;
}
```

### 同步

上面两步获得了Audio_Clock和Video_Clock，这样我们就有了视频流中Frame的显示时间，并且得到了作为基准时间的音频播放时长Audio clock ，可以将视频同步到音频了。

1. 用当前帧的PTS - 上一播放帧的PTS得到一个延迟时间
2. 用当前帧的PTS和Audio_Clock进行比较，来判断视频的播放速度是快了还是慢了
3. 根据2的结果，设置播放下一帧的延迟时间

```c
#define AV_SYNC_THRESHOLD 0.01 // 同步最小阈值
#define AV_NOSYNC_THRESHOLD 10.0 //  不同步阈值
double actual_delay, delay, sync_threshold, ref_clock, diff;

// 当前Frame时间减去上一帧的时间，获取两帧间的延时
delay = vp->pts - is->frame_last_pts;
if (delay <= 0 || delay >= 1.0) { 
    // 延时小于0或大于1秒（太长）都是错误的，将延时时间设置为上一次的延时时间
    delay = is->frame_last_delay;
}

// 获取音频Audio_Clock
ref_clock = get_audio_clock(is);
// 得到当前PTS和Audio_Clock的差值
diff = vp->pts - ref_clock;

sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;

// 调整播放下一帧的延迟时间，以实现同步
if (fabs(diff) < AV_NOSYNC_THRESHOLD) {
    if (diff <= -sync_threshold) { // 慢了，delay设为0
        delay = 0;
    } else if (diff >= sync_threshold) { // 快了，加倍delay
        delay = 2 * delay;
    }
 }
is->frame_timer += delay;
// 最终真正要延时的时间
actual_delay = is->frame_timer - (av_gettime() / 1000000.0);
if (actual_delay < 0.010) {
    // 延时时间过小就设置个最小值
    actual_delay = 0.010;
}
// 根据延时时间刷新视频
schedule_refresh(is, (int) (actual_delay * 1000 + 0.5));
```

### 最后

将视频同步到音频上实现音视频同步基本完成，总体就是动态的过程快了就等待，慢了就加速，在一个你追我赶的状态下实现同步播放。

后面的博客会真正实现一个音视频同步的播放器。