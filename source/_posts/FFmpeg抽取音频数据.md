---
title: FFmpeg抽取音频数据
date: 2018-11-28 16:39:17
categories: 
- 音视频
tags:
- 音视频
- FFmpeg
---



今天开始撸代码，使用FFmpeg的API抽取一个MP4文件的音频数据。

## IDE

应该是第一次在Mac上做C/C++开发，纠结过后选择使用CLion 开发。[CLion](https://www.jetbrains.com/clion/)是 JetBrains下专门用来开发C/C++的IDE，已经用习惯了Android studio和IntelliJ IDEA ，所以CLion用起来还是很顺手的。



在新建一个C项目后，需要把FFmpeg的库导入才能正常运行。我们修改项目的CMakeLists.txt文件。

![](FFmpeg抽取音频数据/extr_voice.png)

