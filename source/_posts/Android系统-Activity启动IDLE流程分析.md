---
title: Android系统 Activity启动IDLE流程分析
date: 2021-07-21 09:22:20
tags:
---

9221系统启动时，tvlauncher出现两次`"ActivityRecord idle"`流程，在该函数中会执行内存回收操作，所以对该流程进行分析，确认IDLE流程是否会对开机速度有影响。

## IdleHandler

在流程分析之前，首先理解下基础概念，什么是`IdleHandler`？







