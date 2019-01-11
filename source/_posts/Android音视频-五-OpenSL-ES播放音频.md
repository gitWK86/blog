---
title: Android音视频(五) OpenSL ES播放音频
date: 2019-01-11 10:36:31
categories: 
- Android音视频
tags:
- 音频
---

[Android音视频(一) Camera2 API采集数据](https://david1840.github.io/2019/01/04/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%B8%80-Camera2-API%E9%87%87%E9%9B%86%E6%95%B0%E6%8D%AE/)

[Android音视频(二)音频AudioRecord和AudioTrack](https://david1840.github.io/2019/01/06/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%BA%8C-%E9%9F%B3%E9%A2%91AudioRecord%E5%92%8CAudioTrack/)

[Android音视频(三)FFmpeg Camera2推流直播](https://david1840.github.io/2019/01/07/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E5%9B%9B-FFmpeg-Camera2%E6%8E%A8%E6%B5%81%E7%9B%B4%E6%92%AD/)

[Android音视频(四)MediaCodec编解码AAC](https://david1840.github.io/2019/01/08/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%B8%89-MediaCodec%E7%A1%AC%E7%BC%96%E7%A1%AC%E8%A7%A3/)

OpenSL ES (Open Sound Library for Embedded Systems)是无授权费、跨平台、针对嵌入式系统精心优化的硬件音频加速API。它为嵌入式移动多媒体设备上的本地应用程序开发者提供标准化, 高性能,低响应时间的音频功能实现方法，并实现软/硬件音频性能的直接跨平台部署，降低执行难度，促进高级音频市场的发展。简单来说OpenSL ES是一个嵌入式跨平台免费的音频处理库。 

在Android中一般使用AudioRecord、MediaRecorder对音频进行采集,使用MediaPlayer、AudioTrack、SoundPool进行音频播放。但这些都是在Java层上的接口，如果使用FFmpeg在C/C++层做音视频处理，那么调用这几个方法就比较麻烦了，所以Android NDK也提供了一个叫做OpenSL的C语言引擎用于声音的处理，这篇博客就是简单使用OpenSL去播放音频。

## 开发流程

OpenSL ES 的开发流程主要有如下6个步骤：

**1、 创建接口对象**

**2、设置混音器**

**3、创建播放器（录音器）**

**4、设置缓冲队列和回调函数**

**5、设置播放状态**

**6、启动回调函数**

其中第4步和第6步是OpenSL ES 播放PCM等数据格式的音频是需要用到的。

## 代码实现





如有问题欢迎留言，[Github源码-AudioDemo](https://github.com/David1840/AudioDemo)  