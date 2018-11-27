---
title: FFmpeg常用结构体分析
date: 2018-11-23 19:58:59
categories: 
- 音视频
tags:
- 音视频
- FFmpeg
---



今天先来了解下FFmpeg中几个我们常用的结构体，防止我们在后面看代码、写代码的时候一脸懵逼。



## AVFormatContext

AVFormatContext是音视频数据,也就是音视频文件的一种抽象和封装，该结构体中包含了多路流，包括音频流、视频流、字幕流等，是FFmpeg中一个贯穿全局的数据结构，很多函数都要以它为参数。 

结构体定义如下（主要参数）：

```c
typedef struct AVFormatContext {
    struct AVInputFormat *iformat; //输入容器格式,用于分流,通过avformat_open_input()设置
    struct AVOutputFormat *oformat; //输出容器格式,用于混流,必须在avformat_write_header()调用前设置
    AVIOContext *pb;  // I/O 上下文
    unsigned int nb_streams; // 流的总数
    AVStream **streams; //所有流的列表,由avformat_new_stream()创建新的流
    int64_t duration; //流的时长
    int64_t bit_rate; //流的比特率
    int64_t probesize; //从指定容器格式的输入中读取最大数据的大小,要足够起播首帧画面
    int64_t max_analyze_duration; //从指定容器格式的输入中读取的最大数据时长
    enum AVCodecID video_codec_id; // 视频的codec_id
    enum AVCodecID audio_codec_id; // 音频的codec_id
    enum AVCodecID subtitle_codec_id; // 字幕的codec_id
    unsigned int max_index_size; // 每条流的最大内存字节数
    unsigned int max_picture_buffer; //从设备获取的实时帧缓冲的最大内存大小
    AVDictionary *metadata; // 整个文件的元数据
    。。。 实在太多了，以后再慢慢了解吧
}AVFormatContext;
```



## AVInputFormat

FFmpeg的解复用器对象，是类似COM接口的数据结构，表示输入文件容器格式，一个文件容器格式对应一个AVInputFormat结构，在程序运行时有多个实例。

## AVStream

是存储每一个视频/音频流信息的结构体，位于avoformat.h文件中。使用解复用器从容器中解析出不同的流，在FFmpeg中流的对象就是AVStream，保存在AVFormatContext的streams数组中。

## AVCodecContext

描述编解码器上下文的数据结构，包含众多编解码器需要的参数信息。

## AVPacket

FFmpeg使用AVPacket来存放编码后的视频帧数据，AVPacket保存了解复用之后、解码之前的数据（仍然是压缩后的数据）和关于这些数据的一些附加信息。AVPacket实际上可用做一个容器，它本身不包含压缩的媒体数据，而是通过data指针引用数据的缓存空间。

## AVCodec

存储编解码器信息的结构体

## AVFrame

用来描述解码出的音视频数据，必须使用av_frame_alloc分配，av_frame_free释放

## AVIOContext

文件操作的顶层结构，实现了带缓冲的读写操作

## URLProtocol

是FFmpeg操作文件的结构，包括open、close、read、write、seek等操作