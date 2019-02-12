---
title: Android音视频(五) OpenSL ES录制、播放音频
date: 2019-02-11 10:36:31
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

在Android中一般使用AudioRecord、MediaRecorder对音频进行采集,使用MediaPlayer、AudioTrack、SoundPool进行音频播放。但这些都是在Java层上的接口，如果使用FFmpeg在C/C++层做音视频处理，那么调用这几个方法就比较麻烦了，所以Android NDK也提供了一个叫做OpenSL的C语言引擎用于声音的处理，这篇博客就是简单使用OpenSL去录制、播放音频。

## 开发流程

OpenSL ES 的开发流程主要有如下6个步骤：

**1、创建接口对象**

**2、设置混音器**

**3、创建播放器（录音器）**

**4、设置缓冲队列和回调函数**

**5、设置播放状态**

**6、启动回调函数**

其中第4步和第6步是OpenSL ES 播放PCM等数据格式的音频是需要用到的。

## 代码实现

### 定义Native方法
``` java

    //播放音频
    public native int play(String filePath);

    //停止播放音频
    public native int playStop();
    
    //录制音频
    public native int record(String filePath);
    
    //停止录制音频
    public native int stopRecod();
    
```

### 录音

#### 参数配置
``` c
    //设置IO设备（麦克风）
    SLDataLocator_IODevice io_device = {
            SL_DATALOCATOR_IODEVICE,         //类型 这里只能是SL_DATALOCATOR_IODEVICE
            SL_IODEVICE_AUDIOINPUT,          //device类型  选择了音频输入类型
            SL_DEFAULTDEVICEID_AUDIOINPUT,   //deviceID 对应的是SL_DEFAULTDEVICEID_AUDIOINPUT
            NULL                             //device实例
    };
    SLDataSource data_src = {
            &io_device,                      //SLDataLocator_IODevice配置输入
            NULL                             //输入格式，采集的并不需要
    };

    //设置输出buffer队列
    SLDataLocator_AndroidSimpleBufferQueue buffer_queue = {
            SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE,    //类型 这里只能是SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE
            2                                           //buffer的数量
    };
    //设置输出数据的格式
    SLDataFormat_PCM format_pcm = {
            SL_DATAFORMAT_PCM,                             //输出PCM格式的数据
            1,                                             //输出的声道数量
            SL_SAMPLINGRATE_44_1,                          //输出的采样频率，这里是44100Hz
            SL_PCMSAMPLEFORMAT_FIXED_16,                   //输出的采样格式，这里是16bit
            SL_PCMSAMPLEFORMAT_FIXED_16,                   //一般来说，跟随上一个参数
            SL_SPEAKER_FRONT_LEFT,  //双声道配置，如果单声道可以用 SL_SPEAKER_FRONT_CENTER
            SL_BYTEORDER_LITTLEENDIAN                      //PCM数据的大小端排列
    };
    SLDataSink audioSink = {
            &buffer_queue,                   //SLDataFormat_PCM配置输出
            &format_pcm                      //输出数据格式
    };

```
#### 录音流程

```
    //1 创建引擎
    SLEngineItf eng = CreateRecordSL();
    if (eng) {
        LOGE("CreateSL success！ ");
    } else {
        LOGE("CreateSL failed！ ");
    }


    //创建录制的对象，并且指定开放SL_IID_ANDROIDSIMPLEBUFFERQUEUE这个接口
    const SLInterfaceID id[1] = {SL_IID_ANDROIDSIMPLEBUFFERQUEUE};
    const SLboolean req[1] = {SL_BOOLEAN_TRUE};
    re = (*eng)->CreateAudioRecorder(eng,        //引擎接口
                                     &recorder_object,   //录制对象地址，用于传出对象
                                     &data_src,          //输入配置
                                     &audioSink,         //输出配置
                                     1,                  //支持的接口数量
                                     id,                 //具体的要支持的接口
                                     req                 //具体的要支持的接口是开放的还是关闭的
    );

    if (re != SL_RESULT_SUCCESS) {
        LOGE("CreateAudioRecorder failed!");
        return -1;
    }

    //实例化这个录制对象
    re = (*recorder_object)->Realize(recorder_object, SL_BOOLEAN_FALSE);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("Realize failed!");
    }

    //获取录制接口
    re = (*recorder_object)->GetInterface(recorder_object, SL_IID_RECORD, &recordItf);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("GetInterface1 failed!");
    }
    //获取Buffer接口
    re = (*recorder_object)->GetInterface(recorder_object, SL_IID_ANDROIDSIMPLEBUFFERQUEUE,
                                          &recorder_buffer_queue);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("GetInterface2 failed!");
    }

    //申请一块内存，注意RECORDER_FRAMES是自定义的一个宏，指的是采集的frame数量，具体还要根据你的采集格式(例如16bit)计算
    pcm_data = malloc(BUFFER_SIZE_IN_BYTES);

    //设置数据回调接口bqRecorderCallback，最后一个参数是可以传输自定义的上下文引用
    re = (*recorder_buffer_queue)->RegisterCallback(recorder_buffer_queue, bqRecorderCallback, 0);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("RegisterCallback failed!");
    }
    //设置录制器为录制状态 SL_RECORDSTATE_RECORDING
    re = (*recordItf)->SetRecordState(recordItf, SL_RECORDSTATE_RECORDING);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("SetRecordState failed!");
    }
    //在设置完录制状态后一定需要先Enqueue一次，这样的话才会开始采集回调
    re = (*recorder_buffer_queue)->Enqueue(recorder_buffer_queue, pcm_data, 8192);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("Enqueue failed!");
    }

```

#### 回调函数
```
//数据回调函数
void bqRecorderCallback(SLAndroidSimpleBufferQueueItf bq, void *context) {

    fwrite(pcm_data, BUFFER_SIZE_IN_BYTES, 1, gFile);
    //取完数据，需要调用Enqueue触发下一次数据回调
    (*bq)->Enqueue(bq, pcm_data, BUFFER_SIZE_IN_BYTES);

}
```

### 播放
```
    //1 创建引擎
    SLEngineItf eng = CreateSL();
    if (eng) {
        LOGE("CreateSL success！ ");
    } else {
        LOGE("CreateSL failed！ ");
        return -1;
    }

    //2 创建混音器
    SLObjectItf mix = NULL;
    SLresult re = 0;

    re = (*eng)->CreateOutputMix(eng, &mix, 0, 0, 0);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("SL_RESULT_SUCCESS failed!");
        return -1;
    }

    re = (*mix)->Realize(mix, SL_BOOLEAN_FALSE);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("(*mix)->Realize failed!");
        return -1;
    }


    SLDataLocator_OutputMix outmix = {SL_DATALOCATOR_OUTPUTMIX, mix};
    SLDataSink audioSink = {&outmix, 0};

    //3 配置音频信息
    //数据定位器 就是定位要播放声音数据的存放位置，分为4种：内存位置，输入/输出设备位置，缓冲区队列位置，和midi缓冲区队列位置。
    SLDataLocator_AndroidSimpleBufferQueue que = {SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE, 10};
    //音频格式
    SLDataFormat_PCM pcm = {
            SL_DATAFORMAT_PCM,
            1,//    声道数
            SL_SAMPLINGRATE_44_1,
            SL_PCMSAMPLEFORMAT_FIXED_16,
            SL_PCMSAMPLEFORMAT_FIXED_16,
            SL_SPEAKER_FRONT_LEFT,
            SL_BYTEORDER_LITTLEENDIAN //字节序，小端
    };
    SLDataSource ds = {&que, &pcm};


    //4 创建播放器
    SLObjectItf player = NULL;
    SLPlayItf iplayer = NULL;
    SLAndroidSimpleBufferQueueItf pcmQue = NULL;
    const SLInterfaceID ids[] = {SL_IID_BUFFERQUEUE};
    const SLboolean req[] = {SL_BOOLEAN_TRUE};
    re = (*eng)->CreateAudioPlayer(eng, &player, &ds, &audioSink,
                                   sizeof(ids) / sizeof(SLInterfaceID), ids, req);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("CreateAudioPlayer failed!");
    } else {
        LOGE("CreateAudioPlayer success!");
    }
    (*player)->Realize(player, SL_BOOLEAN_FALSE);
    //获取player接口
    re = (*player)->GetInterface(player, SL_IID_PLAY, &iplayer);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("GetInterface SL_IID_PLAY failed!");
    }
    re = (*player)->GetInterface(player, SL_IID_BUFFERQUEUE, &pcmQue);
    if (re != SL_RESULT_SUCCESS) {
        LOGE("GetInterface SL_IID_BUFFERQUEUE failed!");
    }

    //设置回调函数，播放队列空调用
    (*pcmQue)->RegisterCallback(pcmQue, pcmCallBack, 0);

    //5 设置为播放状态
    (*iplayer)->SetPlayState(iplayer, SL_PLAYSTATE_PLAYING);

    //6 启动队列回调
    (*pcmQue)->Enqueue(pcmQue, "", 1);
```

 

#### 回调保存数据
```
 //回调函数
void pcmCallBack(SLAndroidSimpleBufferQueueItf bf, void *contex) {
    static char buf[1024 * 1024] = "";
    if (feof(File) == 0) { //没到结尾
        int len = (int) fread(&buf, 1, 1024, File);
        if (len > 0) {
            // 加入队列
            (*bf)->Enqueue(bf, &buf, len);
        }
    }
}
```

如有问题欢迎留言，[Github源码-AudioDemo-openSLActivity](https://github.com/David1840/AudioDemo)  