---
title: 'FFmpeg的下载,编译和安装'
date: 2018-11-21 11:52:21
categories: 
- 音视频
tags:
- FFmpeg
---

FFmpeg的下载,编译和安装，本文所有操作均在Mac系统下完成。



```
git clone https://git.ffmpeg.org/ffmpeg.git

cd ffmpeg

make && make install 

./configure --cc=/usr/bin/clang --prefix=/usr/local/ffmpeg --enable-gpl --enable-nonfree --enable-libfdk-aac --enable-libx264 --enable-libmp3lame --enable-libx265  --enable-filter=delogo --enable-debug --disable-optimizations --enable-libspeex --enable-videotoolbox --enable-shared --enable-pthreads --enable-version3 --enable-hardcoded-tables --host-cflags= --host-ldflags=

```

上面的指令执行结束后在`/usr/local/ffmpeg`目录下就会生成编译好的FFmpeg库。

![](Mac下FFmpeg的下载-编译和安装/ffmpegInstall.png)


## 补充

```
补充缺少的库
brew install speex
brew install x264
brew install x265
sudo make && make install


可能会出现/usr/local/ffmpeg没有权限，可以自己新建一个并修改所有者

sudo mkdir /usr/local/ffmpeg
sudo chown -R ***: ./ffmpeg

```


OK！Mac下就是这么简单，然后就可以去玩一下FFmpeg的指令了。