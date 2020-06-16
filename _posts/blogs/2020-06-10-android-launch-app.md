---
layout: post_no_cmt
title:  "Launcher 启动 App 的流程"
date:   2020-06-10 01:00:00 +0800
categories: [android]
---

# 前言
由于 Android 系统的快速更新，启动 App 第一个 Activity 的代码最新版本和早期版本比已经变化了很多很多，因此只能记录一个大致的思路，具体代码的实现需要参考具体的 Android 版本

# 相关类说明
- AMS（ActivityManagerService）：直接或间接管理着进程，Task，Activity，Service 的信息，并且担负着进程间通信的C/S 模型的 S 的职责，持有各进程的 AMP，因此也作为进程间通信的枢纽
- ActivityRecord：Activity 的相关信息，包括属于哪个进程，哪个 task，哪个 app，Activity信息（ActivityInfo），Activity 状态（ActivityState：INITIALIZING/RESUMED/PAUSING/PAUSED/STOPPING/STOPPED/FINISHING/DESTROYING/DESTROYED/RESTARTING_PROCESS），包名，进程名，启动模式，用户 id 等信息
- TaskRecord：Task 的相关信息，包括属于哪个 Activity 栈（ActivityStack），Task 里面的 Activity 列表，taskId，root Activity 的 affinity，调用者 uid 和 包名等信息
- ProcessRecord：当前运行的进程的所有信息，里面也包含了Activity 和 Services 的所有信息
- ActivityStack：Activity 栈的相关信息，包括所有 Task 的列表，在 Pausing 的 Activity，最后进入 Paused 的 Activity，当前 Resumed 的 Activity 等信息，无历史的 Activity
- ActivityStackSupervisor：ActivityStack 的辅助管理类
- ActivityDisplay：管理显示和分屏的 ActivityStack

# 从 Launcher 点击启动 App 的流程概要

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200617021441333.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)