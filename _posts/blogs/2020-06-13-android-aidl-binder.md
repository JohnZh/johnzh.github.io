---
layout: post_no_cmt
title:  "时序图：基于 AIDL（Binder) 进程间通信之调用远程服务"
date:   2020-06-13 01:00:00 +0800
categories: [android]
---
本地客户端已经绑定上了远程服务后，他们之间的通信流程如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613193625998.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)

在此之前，还有一个客户端 App 在 Activity 里面调用 bindService 直到 ServiceConnection 接收到回调的流程，内容如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200614143113546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)
