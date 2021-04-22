---
title: Android系统-生成OTA增量升级包
date: 2021-03-17 18:18:39
categories: 
- Android系统
tags:
- Android
- OTA增量
- 系统
---

在这里记录一下系统OTA差量包的生成流程。

在系统完成整编之后：

### 1. make otapackage

make otapackage完成了三件事情

- 重新对system.img文件进行了打包；
- 生成差分资源包，路径为out/target/product/<product-name>/obj/PACKAGING/target_files_intermedias/<product-name>-target_files.zip，差分资源包用于生成整包和差分包；
- 生成OTA整包，路径为out/target/product/<product-name>/<product-name>-ota.zip

全量包

![](Android系统-生成OTA增量升级包/ota1.png)

差分资源包

![](Android系统-生成OTA增量升级包/ota2.png)

将差分资源包拷贝出来，命名为msm8953_64-target_file-A.zip

同样的方式在系统做出修改后，生成差分资源包，拷贝并命名为msm8953_64-target_file-B.zip



### 2.制作OTA升级包

在系统源码`build/tools/releasetools`路径下有制作OTA包的相关工具，执行以下命令生成OTA差量包。

`./build/tools/releasetools/ota_from_target_files -n -i 原资源包 目标资源包 生成的差量包`

例子如下：

`./build/tools/releasetools/ota_from_target_files -n -i /Users/david/Downloads/msm8953_64-target_files-A.zip /Users/david/Downloads/msm8953_64-target_files-B.zip /Users/david/Downloads/system_ota.zip`

