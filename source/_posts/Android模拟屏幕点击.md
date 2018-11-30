---
title: Android模拟屏幕点击
date: 2018-11-30 10:28:20
categories: 
- Android开发
tags:
- Android
---

最近在一个没有触摸屏的Android设备上做开发（无奈脸），结果过程中有一个不可避免的弹窗，没法触控就只能由程序去模拟点击事件了。

也在网上找了一些方法，不是不能用就是需要Root权限什么的。最终使用ProcessBuilder来执行命令行语句，模拟使用ADB中的”adb shell tap x y”来点击屏幕，亲测可行，并且代码很简单。

```Kotlin
private fun click(x:Int,y:Int) {
        val order = listOf("input",
                "tap",
                "" + x,
                "" + y)
        ProcessBuilder(order).start()
 }
```

需要传入要点击点的位置坐标，所以要提前计算好坐标，因为我的是专用的Android设备，不用考虑分辨率适配什么的，所以就这样OK了。。。