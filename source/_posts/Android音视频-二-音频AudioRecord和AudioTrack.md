---
title: Android音视频(二)音频AudioRecord和AudioTrack
date: 2019-01-06 11:19:43
categories: 
- Android音视频
tags:
- 音频
---

[Android音视频(一) Camera2 API采集数据](https://david1840.github.io/2019/01/04/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%B8%80-Camera2-API%E9%87%87%E9%9B%86%E6%95%B0%E6%8D%AE/)

AudioRecord和AudioTrack是Android系统提供的用于实现录音、播放音频的功能类，使用这两个类做音频的采集与播放还是非常简单的。

## AudioRecord

```java
private void startRecorder() {
        try {
            // 1、输出pcm文件
            mAudioFile = new File(Environment.getExternalStorageDirectory().getAbsolutePath() + "/RecorderTest/" +
                    System.currentTimeMillis() + ".pcm");
            mAudioFile.getParentFile().mkdirs();
            mAudioFile.createNewFile();
            mFileOutputStream = new FileOutputStream(mAudioFile);

            // 2、配置AudioRecord
              // 声音来源
            int audioSource = MediaRecorder.AudioSource.MIC;
              // 采样率 Hz
            int sampleRate = 44100;
              // 音频通道的配置 MONO 单声道
            int channelConfig = AudioFormat.CHANNEL_IN_MONO;
              // 返回音频数据的格式 
            int audioFormat = AudioFormat.ENCODING_PCM_16BIT;
              //AudioRecord能接受的最小的buffer大小
            int minBufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat);
            mAudioRecorder = new AudioRecord(audioSource, sampleRate, channelConfig,
                    audioFormat, Math.max(minBufferSize, 2048));
            
            // 3、开始录音
            mAudioRecorder.startRecording();

            
            // 4、一边从AudioRecord中读取声音数据到初始化的buffer，一边将buffer中数据导入数据流，写入文件中
            while (mIsRecording) { // 标志位，是否停止录音
                int read = mAudioRecorder.read(mBuffer, 0, 2048);
                if (read > 0) {
                    mFileOutputStream.write(mBuffer, 0, read);
                    // 也可以在这里对音频数据进行处理，压缩、直播等
                } 
            }
            
            // 5、停止录音，释放资源
            mAudioRecorder.stop();
            mAudioRecorder.release();
            mAudioRecorder = null;
            mFileOutputStream.close();

        } catch (IOException | RuntimeException e) {
            e.printStackTrace();
        } finally {
            if (mAudioRecorder != null) {
                mAudioRecorder.release();
                mAudioRecorder = null;
            }
        }
    }
```

### MediaRecorder 和 AudioRecord

Android SDK 中有两套音频采集的API，分别是：MediaRecorder 和 AudioRecord。

1. MediaRecorder是一个更加上层一点的API，它可以直接把手机麦克风录入的音频数据进行编码压缩（如AMR、MP3等）并存成文件

2. AudioRecord则更接近底层，能够更加自由灵活地控制，可以得到原始的一帧帧PCM音频数据。

如果只是想简单地做一个录音机，录制音频文件，就使用 MediaRecorder，而如果需要对音频做进一步的算法处理、或者采用第三方的编码库进行压缩、以及网络传输、直播等应用，则建议使用 AudioRecord。

## AudioTrack

```java
private void doPaly(File mAudioFile) {
        // 音频流的类型
        // STREAM_ALARM：警告声 
        // STREAM_MUSIC：音乐声
        // STREAM_RING：铃声
        // STREAM_SYSTEM：系统声音，例如低电提示音，锁屏音等
        // STREAM_VOCIE_CALL：通话声
        int streamType = AudioManager.STREAM_MUSIC;
    
        // 采样率 Hz
        int sampleRate = 44100;
        // 单声道
        int channelConfig = AudioFormat.CHANNEL_OUT_MONO;
        
        // 音频数据表示的格式
        int audioFormat = AudioFormat.ENCODING_PCM_16BIT;
    
        // MODE_STREAM：在这种模式下，通过write一次次把音频数据写到AudioTrack中。这和平时通过
        // write系统调用往文件中写数据类似，但这种工作方式每次都需要把数据从用户提供的Buffer中拷贝到
        // AudioTrack内部的Buffer中，这在一定程度上会使引入延时。为解决这一问题，AudioTrack就引入
        // 了第二种模式。
    
        // MODE_STATIC：这种模式下，在play之前只需要把所有数据通过一次write调用传递到AudioTrack
        // 中的内部缓冲区，后续就不必再传递数据了。这种模式适用于像铃声这种内存占用量较小，延时要求较
        // 高的文件。但它也有一个缺点，就是一次write的数据不能太多，否则系统无法分配足够的内存来存储
        // 全部数据。
        int mode = AudioTrack.MODE_STREAM;

        int minBufferSize = AudioTrack.getMinBufferSize(sampleRate, channelConfig, audioFormat);

        AudioTrack audioTrack = new AudioTrack(streamType, sampleRate, channelConfig, audioFormat, Math.max(minBufferSize, 2048), mode);

        FileInputStream mFileInputStream = null;
        try {
            mFileInputStream = new FileInputStream(mAudioFile);
            int read;
            audioTrack.play();
            while ((read = mFileInputStream.read(mBuffer)) > 0) {
                int ret = audioTrack.write(mBuffer, 0, read);
                switch (ret) {
                    case AudioTrack.ERROR_BAD_VALUE:
                    case AudioTrack.ERROR_INVALID_OPERATION:
                    case AudioManager.ERROR_DEAD_OBJECT:
                        palyFaile();
                        break;
                    default:
                        break;
                }
            }
        } catch (RuntimeException | IOException e) {
            e.printStackTrace();
            palyFaile();
        } finally {
            mIsPalying = false;
            if (mFileInputStream != null) {
                closeQuietly(mFileInputStream);
            }
            audioTrack.stop();
            audioTrack.release();
        }
    }
```

### AudioTrack 与 MediaPlayer

在Android中播放声音也是有两套API：MediaPlayer和AudioTrack，两者还是有很大的区别的。

1. MediaPlayer可以播放多种格式的声音文件，例如MP3，AAC，WAV，OGG，MIDI等。MediaPlayer会在framework层创建对应的音频解码器。

2. AudioTrack只能播放已经解码的PCM流，如不需要解码的wav文件。

MediaPlayer在framework层还是会创建AudioTrack，把解码后的PCM数流传递给AudioTrack，AudioTrack再传递给AudioFlinger进行混音，然后才传递给硬件播放,所以是MediaPlayer包含了AudioTrack。



[Demo](https://github.com/David1840/AudioDemo)