---
title: Android音视频(一) MediaRecorder录制视频
date: 2019-01-04 19:27:01
categories: 
- Android音视频
tags:
- Camera2
- 直播
---

这是Android音视频的第一篇文章，终于回到了我的老本行上，后面音视频开发会基于Android平台（关键我也不太会其他平台，后面会慢慢学习。。。）

首先使用Android原生的API去实现一个视频的录制，有关Android Camera2的基本操作、预览等都可以查看之前的博客[Android Camera2开发](https://david1840.github.io/2019/01/04/Android-Camera2%E5%BC%80%E5%8F%91/)

主要看一下MediaRecorder录制视频相关代码

```java
private void startRecordingVideo() {
    if (null == mCameraDevice || !mTextureView.isAvailable() || null == mPreviewSize) {
        return;
    }
    try {
        // 关闭之前的会话，新的会话会添加录像的Target
        closePreviewSession();
        // 配置MediaRecorder，音频、视频来源，编码格式等
        setUpMediaRecorder();
        SurfaceTexture texture = mTextureView.getSurfaceTexture();
        assert texture != null;
        texture.setDefaultBufferSize(mPreviewSize.getWidth(), mPreviewSize.getHeight());
        // 创建一个适合视频录制的请求
        mPreviewBuilder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_RECORD);
        List<Surface> surfaces = new ArrayList<>();

        // Set up Surface for the camera preview
        Surface previewSurface = new Surface(texture);
        surfaces.add(previewSurface);
        mPreviewBuilder.addTarget(previewSurface);

        // Set up Surface for the MediaRecorder 重要的一步，视频信息会交给mMediaRecorder处理
        Surface recorderSurface = mMediaRecorder.getSurface();
        surfaces.add(recorderSurface);
        mPreviewBuilder.addTarget(recorderSurface);

        // Start a capture session
        // Once the session starts, we can update the UI and start recording
        mCameraDevice.createCaptureSession(surfaces, new CameraCaptureSession.StateCallback() {

            @Override
            public void onConfigured(@NonNull CameraCaptureSession cameraCaptureSession) {
                mPreviewSession = cameraCaptureSession;
                updatePreview();
                getActivity().runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        // UI
                        mButtonVideo.setText(R.string.stop);
                        mIsRecordingVideo = true;

                        // 开始录制
                        mMediaRecorder.start();
                    }
                });
            }

            @Override
            public void onConfigureFailed(@NonNull CameraCaptureSession cameraCaptureSession) {
                Activity activity = getActivity();
                if (null != activity) {
                    Toast.makeText(activity, "Failed", Toast.LENGTH_SHORT).show();
                }
            }
        }, mBackgroundHandler);
    } catch (CameraAccessException | IOException e) {
        e.printStackTrace();
    }

}
```



```java
// 配置MediaRecorder
private void setUpMediaRecorder() throws IOException {
    final Activity activity = getActivity();
    if (null == activity) {
        return;
    }
    // 设置要用于录制的音频源。
    mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
    // 设置要用于录制的视频源。
    mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.SURFACE);
    // 设置录制期间生成的输出文件的格式。
    mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4);
    
    // 生成MP4文件路径
    if (mNextVideoAbsolutePath == null || mNextVideoAbsolutePath.isEmpty()) {
        mNextVideoAbsolutePath = getVideoFilePath(getActivity());
    }
    mMediaRecorder.setOutputFile(mNextVideoAbsolutePath);
    
    // 设置用于录制的视频编码比特率。
    mMediaRecorder.setVideoEncodingBitRate(10000000);
    // 设置要捕获的视频的帧速率。
    mMediaRecorder.setVideoFrameRate(30);
    mMediaRecorder.setVideoSize(mVideoSize.getWidth(), mVideoSize.getHeight());
    // 设置要用于录制的视频编码器。
    mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264);
    // 设置要用于录制的音频编码器。
    mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC);
    int rotation = activity.getWindowManager().getDefaultDisplay().getRotation();
    switch (mSensorOrientation) {
        case SENSOR_ORIENTATION_DEFAULT_DEGREES:
            mMediaRecorder.setOrientationHint(DEFAULT_ORIENTATIONS.get(rotation));
            break;
        case SENSOR_ORIENTATION_INVERSE_DEGREES:
            mMediaRecorder.setOrientationHint(INVERSE_ORIENTATIONS.get(rotation));
            break;
    }
    // 在调用start前必须的一步
    mMediaRecorder.prepare();
}
```



```
/**
 * 常规使用MediaRecorder去录制视频的例子如下：
 * MediaRecorder recorder = new MediaRecorder();
 * recorder.setAudioSource(MediaRecorder.AudioSource.MIC);
 * recorder.setOutputFormat(MediaRecorder.OutputFormat.THREE_GPP);
 * recorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
 * recorder.setOutputFile(PATH_NAME);
 * recorder.prepare();
 * recorder.start();   // Recording is now started
 * ...
 * recorder.stop();
 * recorder.reset();   // You can reuse the object by going back to setAudioSource() step
 * recorder.release(); // Now the object cannot be reused
 **/
```



[录像：android-Camera2Video](https://github.com/googlesamples/android-Camera2Video) Google 示例程序，有预览、录像功能，非常好的入门学习代码。