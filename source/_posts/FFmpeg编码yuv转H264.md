---
title: FFmpeg编码yuv转H264
date: 2020-08-18 15:10:57
categories: 
- FFmpeg
tags:
- 音视频
- FFmpeg

---

紧接上一章内容，将视频文件添加一个红色方框后文件转成了YUV数据，这一节就再处理下YUV数据，编码成H.264文件。整体流程也比较简单，源码如下：

```c


#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libavutil/imgutils.h>


int main(int argc, char *argv[]) {
    AVFormatContext *pFormatCtx;
    AVOutputFormat *fmt;
    AVStream *video_st;
    AVCodecContext *pCodecCtx;
    AVCodec *pCodec;
    AVPacket pkt;
    uint8_t *picture_buf;
    AVFrame *pFrame;
    int picture_size;
    int y_size;
    int framecnt = 0;
    FILE *in_file = fopen("/Users/liuwei/Desktop/new_test.yuv", "rb");   //Input raw YUV data
    int in_w = 568, in_h = 320;        //new_test.yuv的宽高
    int framenum = 259;       //Frames to encode,这个值是我将原本的视频文件test.mp4解码为YUV文件时得到的数据
    const char *out_file = "/Users/liuwei/Desktop/new_test.h264";

    //Method1.
    pFormatCtx = avformat_alloc_context();
  
    //Guess Format根据输出文件后缀获取已注册输出格式列表中的输出格式，如果没有匹配的，则返回NULL。
    fmt = av_guess_format(NULL, out_file, NULL);
    if (fmt == NULL){
        return -1;
    }
    pFormatCtx->oformat = fmt;

    //打开输出文件
    if (avio_open(&pFormatCtx->pb, out_file, AVIO_FLAG_READ_WRITE) < 0) {
        printf("Failed to open output file! \n");
        return -1;
    }

    //创建流
    video_st = avformat_new_stream(pFormatCtx, 0);
    if (video_st == NULL) {
        return -1;
    }

    // 根据视频编解码器获取对应的编码器
    pCodec = avcodec_find_encoder(fmt->video_codec);
    if (!pCodec) {
        printf("Can not find encoder! \n");
        return -1;
    }
    // 
    pCodecCtx = avcodec_alloc_context3(pCodec);
    if (!pCodecCtx) {
        fprintf(stderr, "Could not allocate video codec context\n");
        exit(1);
    }


    //设置参数
    pCodecCtx->codec_id = fmt->video_codec;
    pCodecCtx->codec_type = AVMEDIA_TYPE_VIDEO;
    pCodecCtx->pix_fmt = AV_PIX_FMT_YUV420P;
    pCodecCtx->width = in_w;
    pCodecCtx->height = in_h;
    pCodecCtx->bit_rate = 400000; // 码率
    pCodecCtx->gop_size = 250; // 每250帧产生一个关键帧，new_test.yuv只有259帧，表示最后只有2个I帧

    pCodecCtx->time_base.num = 1;
    pCodecCtx->time_base.den = 25; //时间基

    //H264
    //pCodecCtx->me_range = 16;
    //pCodecCtx->max_qdiff = 4;
    //pCodecCtx->qcompress = 0.6;
    pCodecCtx->qmin = 10;
    pCodecCtx->qmax = 51;

    //Optional Param
    pCodecCtx->max_b_frames = 3;

    // Set Option
    AVDictionary *param = 0;
    //H.264
    if (pCodecCtx->codec_id == AV_CODEC_ID_H264) {
        av_dict_set(&param, "preset", "slow", 0); //压缩速度慢，保证视频质量
        av_dict_set(&param, "tune", "zerolatency", 0);//零延迟
    }
    //H.265
    if (pCodecCtx->codec_id == AV_CODEC_ID_H265) {
        av_dict_set(&param, "preset", "ultrafast", 0);
        av_dict_set(&param, "tune", "zero-latency", 0);
    }

    //Show some Information
    av_dump_format(pFormatCtx, 0, out_file, 1);

    if (avcodec_open2(pCodecCtx, pCodec, &param) < 0) {
        printf("Failed to open encoder! \n");
        return -1;
    }


    pFrame = av_frame_alloc();
    //返回使用给定参数存储图像所需的数据量的大小
    picture_size = av_image_get_buffer_size(pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height, 4);
    picture_buf = (uint8_t *) av_malloc(picture_size);
    //分配内存 
    av_image_fill_arrays(pFrame->data, pFrame->linesize, picture_buf, pCodecCtx->pix_fmt, pCodecCtx->width,
                         pCodecCtx->height, 1);

    //Write File Header
    avformat_write_header(pFormatCtx, NULL);

    av_new_packet(&pkt, picture_size);

    y_size = pCodecCtx->width * pCodecCtx->height;

    for (int i = 0; i < framenum; i++) {
        //从文件中读取YUV数据
        if (fread(picture_buf, 1, y_size * 3 / 2, in_file) <= 0) {
            printf("Failed to read raw data! \n");
            return -1;
        } else if (feof(in_file)) {
            break;
        }
        pFrame->data[0] = picture_buf;              // Y
        pFrame->data[1] = picture_buf + y_size;      // U
        pFrame->data[2] = picture_buf + y_size * 5 / 4;  // V
        //PTS
        pFrame->pts = i * (video_st->time_base.den) / ((video_st->time_base.num) * 25);
        //编码
        avcodec_send_frame(pCodecCtx, pFrame);
        int ret = avcodec_receive_packet(pCodecCtx, &pkt);
        if (ret == 0) {
            printf("Succeed to encode frame: %5d\tsize:%5d\n", framecnt, pkt.size);
            framecnt++;
            pkt.stream_index = video_st->index;
            av_write_frame(pFormatCtx, &pkt);
        }
        av_packet_unref(&pkt);
    }


    //Write file trailer
    av_write_trailer(pFormatCtx);

    //Clean
    avcodec_free_context(&pCodecCtx);
    av_free(pFrame);
    av_free(picture_buf);
    avio_close(pFormatCtx->pb);
    avformat_free_context(pFormatCtx);

    fclose(in_file);

    return 0;
}
```

压缩效果非常明显，84M的YUV视频压缩成H264后只有492KB。