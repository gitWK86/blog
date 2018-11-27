---
title: FFmpeg常用结构体、方法简介
date: 2018-11-23 19:58:59
categories: 
- 音视频
tags:
- 音视频
- FFmpeg
---



今天先来了解下FFmpeg中几个我们常用的结构体，防止我们在后面看代码、写代码的时候一脸懵逼。

## 常用结构体

### AVFormatContext

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



### AVInputFormat

FFmpeg的解复用器对象，是类似COM接口的数据结构，表示输入文件容器格式，一个文件容器格式对应一个AVInputFormat结构，在程序运行时有多个实例。

### AVStream

是存储每一个视频/音频流信息的结构体，位于avoformat.h文件中。使用解复用器从容器中解析出不同的流，在FFmpeg中流的对象就是AVStream，保存在AVFormatContext的streams数组中。

### AVCodecContext

描述编解码器上下文的数据结构，包含众多编解码器需要的参数信息。

### AVPacket

FFmpeg使用AVPacket来存放编码后的视频帧数据，AVPacket保存了解复用之后、解码之前的数据（仍然是压缩后的数据）和关于这些数据的一些附加信息。AVPacket实际上可用做一个容器，它本身不包含压缩的媒体数据，而是通过data指针引用数据的缓存空间。

### AVCodec

存储编解码器信息的结构体。

### AVFrame

用来描述解码出的音视频数据，必须使用av_frame_alloc分配，av_frame_free释放。

### AVIOContext

文件操作的顶层结构，实现了带缓冲的读写操作。

### URLProtocol

是FFmpeg操作文件的结构，包括open、close、read、write、seek等操作。



## 常用方法

### av_register_all 

初始化所有组件，只有调用了该函数，才能使用复用器和编解码器(FFmpeg4.0以上被废弃，不推荐使用，可以不调用)。



### avformat_alloc_context

AVFormatContext要用avformat_alloc_context()进行初始化，分配内存空间。



### avformat_open_input

```C
int avformat_open_input(AVFormatContext **ps, const char *url, AVInputFormat *fmt, AVDictionary **options);
```

主要功能是打开一个文件，读取header，不会涉及打开解码器，与之对应的是avformat_close_input函数关闭文件。如果打开文件成功，AVFormatContext  ps就会在函数中初始化完成。



### av_guess_format

```C
AVOutputFormat *av_guess_format(const char *short_name,
                                const char *filename,
                                const char *mime_type);
```



从所编译的ffmpeg库支持的muxer中查找与文件名有关联的容器类型。



### avformat_new_stream

```C
AVStream *avformat_new_stream(AVFormatContext *s, const AVCodec *c);
```

在 AVFormatContext 中创建新的 Stream 流通道。



### avio_open

```C
int avio_open(AVIOContext **s, const char *url, int flags);
```

用于打开FFmpeg的输入/输出文件。



### av_find_best_stream

```C
int av_find_best_stream(AVFormatContext *ic,
                        enum AVMediaType type,
                        int wanted_stream_nb,
                        int related_stream,
                        AVCodec **decoder_ret,
                        int flags);
```

在文件中找到“最好”的用户所期望的流

### av_read_frame

```C
int av_read_frame(AVFormatContext *s, AVPacket *pkt);
```

读取码流中的若干音频帧或者1帧视频。



### av_write_frame

FFmpeg调用`avformat_write_header`函数写头部信息，`av_write_frame`函数写1帧数据，调用`av_write_trailer`写尾部信息。



