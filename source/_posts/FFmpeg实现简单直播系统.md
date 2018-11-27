---
title: FFmpeg实现简单直播系统
date: 2018-11-23 09:48:29
categories: 
- 音视频
tags:
- 音视频
- FFmpeg
---

现在市面上的各种直播应用越来越多，直播相关技术也成为最火热的技术方向之一，所以今天就用FFpmeg来实现一个简单的直播系统。

## 环境搭建

FFmpeg之前也看到它有直播推流和直播拉流的命令，那这个流推到哪里?又要从哪里拉?这个时候就需要我们的关键角色--**流媒体服务器**。

我使用Nginx服务，并配置好RTMP服务。

`brew install nginx-full --with-rtmp-module`

等安装完成后修改配置文件

```
vi /usr/local/etc/nginx/nginx.conf

添加如下:

rtmp {
      server{

            listen 1935; //端口号
            chunk_size 4000;
            
            //指定流应用
            application live
           {
                 live on;
                 record off;
                 allow play all;
           }
      }
}

```

最后重启服务

`nginx -s reload`

## 直播

环境Ok了，接下来就可以试一下推流拉流直播了。

首先可以推一个视频

`ffmpeg -re -i out.mp4 -c copy -f flv rtmp://127.0.0.1:1935/live/room`

使用FFplay播放

`ffplay rtmp://localhost:1935/live/room`

![](FFmpeg实现简单直播系统/ffmpeglive.png)

图片左侧是推流，右侧ffplay拉流，中间为ffplay播放的视频。

然后可以试着直播下我的桌面:

`ffmpeg -f avfoundation -i "1" -vcodec libx264 -preset ultrafast -acodec libfaac -f flv rtmp://127.0.0.1:1935/live/room`

都是同样的原理，很简单，赶紧去试一下吧!
