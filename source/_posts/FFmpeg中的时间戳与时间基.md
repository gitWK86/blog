---
title: FFmpeg中的时间戳与时间基
date: 2018-12-05 13:42:44
categories: 
- FFmpeg
tags:
- 音视频
- FFmpeg
---

最近在学习FFmpeg的过程中发现，其他的知识点还比较清楚，就是对于FFmpeg中的时间基概念模糊，前面做Demo也只是照猫画虎，没有真正理解，所以今天花时间好好理解一下时间戳和时间基的概念。

### PTS和DTS

这两个概念其实在刚开始的时候就提到过，今天的概念跟它们还是有很大关系的，所以再说一次概念，具体解释请看[音视频基础概念](https://www.jianshu.com/p/e3acc140aa90)。

- PTS: Decode Time Stamp，显示渲染用的时间戳，告诉我们什么时候需要显示
- DTS: Presentation Time Stamp，视频解码时的时间戳，告诉我们什么时候需要解码



### 时间基

在写代码处理音视频流的时候经常会看到`in_stream->time_base`这样的代码，这表示的是输入流的时间基，time_base时间基的结构体定义如下：

```C
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

可以看出时间基是一个分数，以秒为单位，比如1/50秒，那它到底表示的是什么意思呢？以帧率为例，如果它的时间基是1/50秒，那么就表示每隔1/50秒显示一帧数据，也就是每1秒显示50帧，帧率为50FPS。

每一帧数据都有对应的PTS，在播放视频或音频的时候我们需要将PTS时间戳转化为以秒为单位的时间，用来最后的展示。那如何计算一桢在整个视频中的时间位置？

```c
static inline double av_q2d(AVRational a){
    return a.num / (double) a.den;
}

//计算一桢在整个视频中的时间位置
timestamp(秒) = pts * av_q2d(st->time_base);

//计算视频长度的方法：
time(秒) = st->duration * av_q2d(st->time_base);
```

#### 内部时间基

FFmpeg中的所有时间都是以它为一个单位

```c
/**
 * Internal time base represented as integer
 * 内部时间基
 */
#define  AV_TIME_BASE            1000000

//内部时间基的分数表示，实际上它是AV_TIME_BASE的倒数
#define  AV_TIME_BASE_Q   (AVRational){1, AV_TIME_BASE}

//ffmpeg内部的时间与标准的时间转换方法
timestamp(ffmpeg内部时间戳) = AV_TIME_BASE * time(秒)
time(秒) = AV_TIME_BASE_Q * timestamp(ffmpeg内部时间戳)
```

当需要把视频跳转到N秒的时候可以使用下面的方法：

```
av_seek_frame(fmt_ctx, index_of_video, N * AV_TIME_BASE, AVSEEK_FLAG_BACKWARD);
```

有时候我们需要在不同的时间基之间做换算。

```c
int64_t av_rescale_q(int64_t a, AVRational bq, AVRational cq) av_const;
```

这个方法实际的操作是 `a * bq / cq`，看起来简单，但它内部处理了数值溢出的问题，所以我们在操作的时候最好还是直接调用这个方法。