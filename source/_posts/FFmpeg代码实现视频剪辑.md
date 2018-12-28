---
title: FFmpeg代码实现视频剪切
date: 2018-12-06 12:07:22
categories: 
- FFmpeg
tags:
- 音视频
- FFmpeg
---

有几天没写FFmpeg代码了，今天趁着有空闲来撸下FFmpeg剪切视频代码，我也是边学习边写，如果有错误，请在评论中指出，互相学习。

## 思路

说起来这个功能的实现也很简单，给定一个起始时间、一个结束时间，把视频文件打开，然后把容器中的每条流从起始时间开始，到结束时间为止的数据拷贝到输出流，然后输出流保存为容器，这样就能看到一个剪切后的视频文件了。

## 代码实现

#### 第一步 定义参数

```c
AVFormatContext *ifmt_ctx = NULL;
AVFormatContext *ofmt_ctx = NULL;
AVOutputFormat *ofmt = NULL;
AVPacket pkt;

double start_seconds； //开始时间
double end_seconds；   //结束时间
const char *in_filename； //输入文件
const char *out_filename；//输出文件
```

#### 第二步 初始化上下文

```c
avformat_open_input(&ifmt_ctx, in_filename, 0, 0);

//本质上调用了avformat_alloc_context、av_guess_format这两个函数，即创建了输出上下文，又根据输出文件后缀生成了最适合的输出容器
avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, out_filename); 
ofmt = ofmt_ctx->oformat;
```

#### 第三步 创建流及参数拷贝

```c
for (i = 0; i < ifmt_ctx->nb_streams; i++) {
        AVStream *in_stream = ifmt_ctx->streams[i];
        AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
        if (!out_stream) {
            fprintf(stderr, "Failed allocating output stream\n");
            ret = AVERROR_UNKNOWN;
            goto end;
        }
        avcodec_parameters_copy(out_stream->codecpar, in_stream->codecpar);
        out_stream->codecpar->codec_tag = 0;
    }
```

#### 第四步 打开输出文件

```c
avio_open(&ofmt_ctx->pb, out_filename, AVIO_FLAG_WRITE);
```

#### 第五步  处理、写入数据

```c
// 写头信息
ret = avformat_write_header(ofmt_ctx, NULL);
if (ret < 0) {
    fprintf(stderr, "Error occurred when opening output file\n");
    goto end;
}

//跳转到指定帧
ret = av_seek_frame(ifmt_ctx, -1, start_seconds * AV_TIME_BASE, AVSEEK_FLAG_ANY);
if (ret < 0) {
        fprintf(stderr, "Error seek\n");
        goto end;
}

// 根据流数量申请空间，并全部初始化为0
int64_t *dts_start_from = malloc(sizeof(int64_t) * ifmt_ctx->nb_streams);
memset(dts_start_from, 0, sizeof(int64_t) * ifmt_ctx->nb_streams);

int64_t *pts_start_from = malloc(sizeof(int64_t) * ifmt_ctx->nb_streams);
memset(pts_start_from, 0, sizeof(int64_t) * ifmt_ctx->nb_streams);

while (1) {
        AVStream *in_stream, *out_stream;

        //读取数据
        ret = av_read_frame(ifmt_ctx, &pkt);
        if (ret < 0)
            break;

        in_stream = ifmt_ctx->streams[pkt.stream_index];
        out_stream = ofmt_ctx->streams[pkt.stream_index];

        // 时间超过要截取的时间，就退出循环
        if (av_q2d(in_stream->time_base) * pkt.pts > end_seconds) {
            av_packet_unref(&pkt);
            break;
        }

        // 将截取后的每个流的起始dts 、pts保存下来，作为开始时间，用来做后面的时间基转换
        if (dts_start_from[pkt.stream_index] == 0) {
            dts_start_from[pkt.stream_index] = pkt.dts;
        }
        if (pts_start_from[pkt.stream_index] == 0) {
            pts_start_from[pkt.stream_index] = pkt.pts;
        }

        // 时间基转换
        pkt.pts = av_rescale_q_rnd(pkt.pts - pts_start_from[pkt.stream_index], in_stream->time_base, out_stream->time_base, AV_ROUND_NEAR_INF | AV_ROUND_PASS_MINMAX);
        pkt.dts = av_rescale_q_rnd(pkt.dts - dts_start_from[pkt.stream_index], in_stream->time_base,out_stream->time_base, AV_ROUND_NEAR_INF | AV_ROUND_PASS_MINMAX);
        
        if (pkt.pts < 0) {
            pkt.pts = 0;
        }
        if (pkt.dts < 0) {
            pkt.dts = 0;
        }

        pkt.duration = (int) av_rescale_q((int64_t) pkt.duration, in_stream->time_base, out_stream->time_base);
        pkt.pos = -1;

        //一帧视频播放时间必须在解码时间点之后，当出现pkt.pts < pkt.dts时会导致程序异常，所以我们丢掉有问题的帧，不会有太大影响。
        if (pkt.pts < pkt.dts) {
            continue;
        }
    
        ret = av_interleaved_write_frame(ofmt_ctx, &pkt);
        if (ret < 0) {
            fprintf(stderr, "Error write packet\n");
            break;
        }

        av_packet_unref(&pkt);
    }

//释放资源
free(dts_start_from);
free(pts_start_from);

//写文件尾信息
av_write_trailer(ofmt_ctx);
```



整个处理流程就这样了，还是比较简单的。