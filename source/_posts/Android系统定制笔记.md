---
title: Android系统定制笔记
date: 2020-07-30 19:22:56
categories: 
- Android系统
tags:
- Android
- 系统
---

## 一、fastboot刷机

首先安装adb和fastboot

`sudo apt-get install android-tools-adb android-tools-fastboot`

### 进入Fastboot模式

首先，确保你的手机能够adb连接，然后通过adb执行如下指令进入Fastboot模式，命令如下：

```
sudo adb reboot-bootloader
```

稍等片刻，手机会重启进入Fastboot模式，查看通过如下命令进行确认：

```
sudo fastboot devices
```

### 刷img文件

1. 刷boot.img指令

```
sudo fastboot flash boot boot.img
```

2. 刷system.img指令

```
sudo fastboot flash system system.img
```

3. 刷userdata.img指令

```
sudo fastboot flash userdata userdata.img
```

4. 重启手机即可

```
sudo fastboot reboot
```

## 二、系统编译

### 编译系统-全编

#### 1.安装软件

```
sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev libc6-dev
lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev libgl1-mesa-dev g++-multilib mingw32
tofrodos python-markdown libxml2-utils xsltproc
```

#### 2.安装openJDK

```
sudo add-apt-repository ppa:openjdk-r/ppa 
sudo apt-get update
sudo apt-get install openjdk-8-jdk 
```

如果你已经安装openJDK7或其他，可以使用下面命令修改

```
sudo update-alternatives --config java 
```

#### 3.编译系统

```
1、source build/envsetup.sh
2、lunch
  //该命令会显示可编译的所有版本，请选择一种，输入对应数字即可
3、make –j8
```

然后请等待编译结束，编译完成后在/out/target/product/msm8953_64/下找到对应的img文件

### 编译系统-模块编译

```
make update-api //更新API接口，代码有修改，git pull拉取代码后请先执行该命令

make systemimage -j8  //单独编译system.img 
```

#### 单模块编译

##### 修改应用源码

例如修改了设置Settings代码，可以单独编译Settings的源码，编译出Settings.apk验证

```
cd packages/apps/Settings/

mm // 单独编译Settings.apk

```

编译完成后 

```
adb root
adb remount
adb push Settings.apk /system/priv-app/Settings/
adb reboot
```

#####  修改源码framework后编译

1. framework/base/core/res/res下添加或修改资源文件后需要先编译资源，然后编译framework 才可正常引用。

```
cd frameworks/base/core/res/ 
mm
```

2. 编译 framework.jar 

```
cd frameworks/base/ 
mm 
```

3. 如果 frameworks/base/services 下有修改，则要编译frameworks/base/services/java/ 执行mm ，编译 services.jar

4. 执行如下命令

```
adb remount
adb push framework-res.apk /system/framework/
adb push framework.jar /system/framework/
adb push services.jar /system/framework/ （如果有修改的话）
```

5. push后，可以cd system/framework 进入目录，以ll命令确认下是否push成功。
6. adb reboot 重启设备。

## 三、系统定制

### 1 Launcher过滤App

LauncherMode.java 中有一个loadAllApps函数，Launcher在其中加载所有App

```java
// Create the ApplicationInfos
for (int i = 0; i < apps.size(); i++) {
    LauncherActivityInfoCompat app = apps.get(i);
    Log.e(TAG,"loadAllPackages ="+app.getComponentName().getPackageName());
    if(app.getComponentName().getPackageName() == "com.android.settings" || app.getComponentName().getPackageName().contains("decard")){
        Log.e(TAG,"settings || decard");
        // This builds the icon bitmaps.
        mBgAllAppsList.add(new AppInfo(mContext, app, user, mIconCache, quietMode));
    }
}
```

### 2 去除抽屉

[Android7.1 Launcher3去除抽屉](https://david1840.github.io/2020/08/07/Android7-1-Launcher3%E5%8E%BB%E9%99%A4%E6%8A%BD%E5%B1%89/)

### 3 添加自定义系统服务

[ Android 添加自定义系统服务](https://david1840.github.io/2020/08/07/Android-%E6%B7%BB%E5%8A%A0%E8%87%AA%E5%AE%9A%E4%B9%89%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1/)

