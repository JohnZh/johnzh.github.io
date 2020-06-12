---
layout: post_no_cmt
title:  "DiskLruCache 源码分析图"
date:   2017-05-10 01:00:00 +0800
categories: [android]
---
![DiskLruCache](https://img-blog.csdnimg.cn/2020061213111171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)

由于在缓存读写过程中涉及到 io 操作，且在 lru 策略发生的时候，即 remove 的时候会导致 cache 其他操作的阻塞，因此在使用中务必是多线程，务必要脱离主线程