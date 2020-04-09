---
layout: post
title:  "Socket、TCP、UDP 碎片记录"
date:   2020-02-02 00:00:00 +0800
categories: [network]
---

# 什么是 Socket
封装了 TCP、UDP 传输层协议操作的抽象层，编程中即为提供了传输层协议 TCP、UDP 操作的 API

# TCP 和 UDP 简单比较
都是传输层协议。TCP 面向连接，面向字节流，有流量控制和差错控制；UDP 面向无连接，面向报文，没有控制手段。因此 TCP 比 UDP 在传输中更可靠，但是传输效率相对低

## 面向字节流和面向报文

### 面向字节流
数据似流水，缓冲区似水池。发送方 write 时候不停的向着池子里面放数据，可以一次写完也可以分次（10 字节，50 字节，100 字节）写完，接收方 read 时候不停的向接受方缓存区里面读数据，也可以一次读完或者多次读完。发送方和接收方似放水和接水。

### 面向报文
发送的是完整报文或者是数据块，发送方 write 一次，接收方就要 read 一次，发送太长，数据下到 ip 层可能会分片，不论发送或者接受，如果超过缓存区大小，多出来的会被丢弃

## 应用
基于 TCP，UDP的优缺点：TCP 可靠安全；UDP 控制少速度快

应用 | 应用层协议 | 传输层协议
---|---|---
万维网|HTTP|TCP
电子邮件|STMP|TCP
远程终端|TELNET|TCP
文件传输|FTP|TCP
域名 IP 转换系统|DNS|UDP
路由选择协议|RIP|UDP
IP 地址分配|BOOTP，DHCP|UDP
网络电话|VOIP 等|UDP
网络视频|RTSP（依赖 RTP&RTCP）|UDP（特殊情况可改用 TCP）

#### 总结
- UDP：一般对实时性要求比较高，数据完整性不是非常高的场景，网络通话视频
- TCP：一般对安全性，可靠性，数据完整性要求比较高的场景

#### 协议补充说明
- 应用层协议，一般都是基于传输层协议 TCP 或者 UDP，如 Http基于 TCP，而传输层协议都是基于网络层 IP 协议
- 网络七层模型：应用层、表示层、会话层、传输层、网络层、链路层、物理层
- TCP/IP 4层模型：应用层（应用层、表示层、会话层）、传输层、网络层、网络接口层（链路层、物理层）


## 攻击

- TCP 有确认机制，三次握手，因此也容易被人利用，例如 DOS，DDOS，CC 等攻击
- UPD 没有确认机制，因此攻击模式就会比较少，但并不是没有，例如 UDP 洪水攻击（UDP FLOOD ATTACK）：大量数据包使得接收方网络堵塞

## TCP 三次握手和四次挥手
### 握手
1. Client --- SNY J ---> Server
2. Client <--- SNY K, ACK J + 1 --- Server
3. Client --- ACK K + 1 ---> Server

### 挥手
1. Client --- FIN M ---> Server
2. Client <--- ACK M + 1 --- Server
3. Client <--- FIN N --- Server
4. Client --- ACK N + 1 ---> Server