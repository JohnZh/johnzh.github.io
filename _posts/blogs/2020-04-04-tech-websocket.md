---
layout: post
title:  "Websocket 介绍，握手，帧结构，关闭，安全"
date:   2020-04-04 00:00:00 +0800
categories: [network]
---

# 背景
简单的说，Websocket 是为了解决 web app 客户端和服务端之间的双向通信的问题而产生的。类似于游戏，即时信息类，实时资讯类软件。

在没有 Websocket 等类似协议之前，web app 双向通信都是采用 http 的  poll 即轮询。其两种形式：

- 周期性请求，获取最新结果
- 请求后阻塞保持长连接，直到等到结果后再次发出请求

产生的几个问题：

- 服务器连接数会大量增加
- 带宽的浪费，由于 HTTP 的无状态，每个 HTTP 请求都可能会有重复信息， HTTP HEADER 就是其中最严重的
- 对于客户端来说，实时性不高，且为了追踪回复还要面临管理着多个请求连接和单个响应连接的关系

而 Websocket 本质上就是为了双向通信而使用单独的一条 TCP 连接

# 介绍
Websocket 分成两部分：握手 + 数据传输

## URIs
- ws://host[:port]/path?query
- wss://host[:port]/path?query

wss 是基于 TLS 的 Websocket

port 端口号是可选项，默认会使用 http/https 的 80 和 443

path 和 query 也不是必须的

```
ws://example.com/chat
wss://secure.example.com/chat
```

## 握手
Websocket 和 HTTP 都是基于 TCP 的应用层协议，两个协议唯一的关系就是 Websocket 握手过程使用了 HTTP Upgrade 请求

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```
服务端愿意建立 Websocket 连接会响应

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### 握手 HTTP 请求：
- GET 请求要求 HTTP 版本大于等于 1.1 
- Host，Upgrade，Connection，Sec-WebSocket-Key，Sec-WebSocket-Version 为必须项，其他 HEADER 为 Optional
- Origin 这个 HEADER 一般情况下是使用在浏览器客户端，缺少这个 HEADER 的情况，服务端不应该认为其来着浏览器客户端。而且该域的作用是为了避免浏览器对 Websocket 服务器的未授权跨域使用
- Sec-WebSocket-Key 128 位随机数，并且使用 base64 编码。用于每次连接时服务端验证合法的 Websocket 客户端。对应 Sec-WebSocket-Accept
- Sec-WebSocket-Protocol 子协议列表。请求头里可以出现多个键值对，也可以只出闲一次，值用逗号分开，但是响应头里面只能有一个键值，为了确定使用哪个子协议。这个字段仅仅是客户端和服务端的协商字段
- Sec-WebSocket-Version 当前版本必须是 13，其他版本已经不再用了

### 握手 HTTP 响应：
- 握手完成必须是 HTTP 101 Switching Protocols，其他状态码都代表握手未完成
- 和请求一样，Upgrade，Connection 都是必须的，对应的还有Sec-WebSocket-Accept
- Sec-WebSocket-Accept 代表服务端是否愿意接受连接。出现这个字段，值必须为 Sec-WebSocket-Key 连同 GUID 的 hash 值，客户端会校验这个值
- Sec-WebSocket-Protocol 一样是 Optional。如果出现，值必然是客户端请求列表里的值，缺省代表 null
- 状态码不是 101，Sec-WebSocket-Accept 丢失，值校验不对都表示连接没有建立，Websocket 帧不会发送

#### Sec-WebSocket-Accept 的校验
服务端处理：
Sec-WebSocket-Key 拼接上 "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" 字符串，然后进行sha1 hash，最后进行 base64 编码

```
Sec-WebSocket-Accept = base64(sha1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```

> rfc6455 section 1.3/section 4 有详细的握手和相关字段的说明
> - [section 1.3 Opening Handshake](https://tools.ietf.org/html/rfc6455#section-1.3)
> - [section 4 Opening Handshake](https://tools.ietf.org/html/rfc6455#section-4)

## 数据传输

传输最小单元是帧 frame，数据传输使用单个至一串帧(也称消息)

为了避免中介和中间代理，客户端发送的数据都进行了掩码处理，而服务端发送的数据没有，这和是否运行在 TLS 上无关。

当服务端收到无掩码处理的过的帧，或者客户端收到来自掩码处理过的帧，这都会导致连接的关闭，服务器会发送带有状态码 1002 协议错误 Close 帧
，而客户端会使用状态码 1002，以此来关闭连接。

### 基础帧结构

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

- FIN 1 bit: 是否为消息的最后一个片段
- RSV1, RSV2, RSV3, 每个 1 bit: 一般都为 0，服务端协商采用扩展时候可以非 0，如果收到非 0 值，但是又无扩展定义这些非 0 值的意义，连接错误
- opcode 4 bit: 定义 Payload data 解释，无效的 opcode 连接会断开
    -  %x0 连续帧，整个消息为分片消息，此帧为多帧中的一个
    -  %x1 文本帧
    -  %x2 二进制帧
    -  %x3-7 保留作为以后使用的非控制类帧
    -  %x8 连接关闭
    -  %x9 ping 操作
    -  %xA pong 操作
    -  %xB-F 保留作为以后使用的控制类帧
- MASK 1 bit: 是否对 Payload data 进行了掩码处理。值 1，存在 masking key，这是用于反向掩码操作使用的。所有客户端发送的帧该字段都是 1
- Payload length, Extended payload length, Extended payload length continued: 7 bits, 7+16 bits, or 7+64 bits: 
    - Payload length 范围在 [0-125]，Payload length 即 Payload data 的长度
    - Payload length = 126, Extended payload length 的 16 bits 无符号整数表示 Payload data 长度
    - Payload length = 127，Extended payload length continued 的 64 bits 无符号整数表示 Payload data 长度
    - 注意，多字节表示长度，使用的是[网络字节序(大端)](#byte_order)
-  Masking-key 0 或 4 字节，32-bit 值
-  Payload data x+y 字节，包括 Extension data + Application data
-  Extension data x 字节: 一般为 0 字节，除非扩展被协商。扩展数据的长度，或者计算方法，如何使用必须在握手时候就协商好。如果扩展数据存在，长度会被包括到 Payload length 里面
-  Application data y 字节，在扩展数据之后，其长度为 Payload lenght 减去 Extension data 的长度

> 更多细节 [rfc6455 section 5.2  Base Framing Protocol](#https://tools.ietf.org/html/rfc6455#section-5.2)

### 掩码处理
掩码算法：本质上就是一个复杂一点的异或运算及其逆运算

```
j == i xor 2;
j xor 2 == i;
```
Websocket 掩码算法：

```
original-octet-i // 原始数据第 i 个字节
transformed-octet-i // 掩码处理后的数据第 i 个字节
masking-key-octet-j // masking-key 的第 j 个字节
j = i mod 4
transformed-octet-i = original-octet-i XOR masking-key-octet-j
或者
transformed-octet-i = original-octet-i XOR masking-key-octet-(i mod 4)
```
反向掩码算法也是如此

> 细节参考 [rfc6455 section 5.3  Client-to-Server Masking](#https://tools.ietf.org/html/rfc6455#section-5.3)

### 安全性
和 Sec-WebSocket-Key & Sec-WebSocket-Accept 以及 Origin 所涉及到安全性的考量一样，掩码处理的安全性目的也并非数据的安全性。

- Sec-WebSocket-Key/Sec-WebSocket-Accept只是表示这是来自客户端合法的 Websocket 请求以及验证服务端完成握手，算法是公开的且简单的，并无任何安全性措施
- Origin，仅用于浏览器的同源策略模型，阻止非同源无授权跨域使用，即服务端可以拒绝未授权的 Origin，但也并非是为了数据安全性
- 掩码算法，掩码算法是公开的，因此虽然数据被掩码处理了，但是也并非是因为数据安全性而做，主要目的是为了避免代理缓存投毒攻击(proxy cache poisioning attack)。

#### 代理缓存投毒攻击
代理缓存投毒攻击的大致步骤：
1. attacker 通过 proxy 服务器向 attcker.com 服务器发出 websocket 连接请求；服务器返回响应，请求结束，Websocket 连接建立，attcker 可以直接发送数据到 attacker.com
2. - attacker 使用 http 报文格式创建了文本，且伪造 host 和资源 比如 host:target.com，资源 /stript.js
   - 通过 Websocket 连接发送，经过 proxy 服务器，服务器认为这是新的 http 请求，消息继续到达 attacker.com 服务器
   - attacker.com 服务器配合 attacker，使用 http 响应报文格式发送消息，且带着 "毒药" 脚本 script.js*，再次经过 proxy 服务器，服务器认为这是新请求的响应，缓存了 host:attacker.com 和资源 /script.js*
3. 之后正常用户访问 target.com/script.js，http 请求到达 proxy 服务器的时候，cache hit，script.js* 返回给了正常用户
4. 掩码处理的目的就是为了让 attcker 无法控制传输文本表现形式，从而让 proxy 服务器产生错误解析，以为是 http 请求


## 连接保持
Websocket 在长时间无数据传输却要保持连接的情况，需要 "心跳" 机制
- 发送方发送：ping [opcode 0x9]
- 接收方响应：pong [opcode 0xA]

ping 帧可能包含 Application data，响应的 pong 帧必须有相同的 Application data

## 关闭握手

### Close 帧 
Close 帧可能包含 "Application data"，用于表示关闭原因：如
- 端点关闭
- 端点收到的帧过大
- 端点收到帧的格式不符合端点的要求

如果帧有 "Application data"，那么最开始的 2 个字节必须是一个无符号的整数(大端序)，代表 [status codo](https://tools.ietf.org/html/rfc6455#section-7.4)。其他的字节可能包含 UTF-8 编码的数据，值内容是关闭的原因。这个原因的内容也许不是人类可读内容，所以客户端需要解析并显示出其内容。

#### 关闭握手
一个端点接收到 Close 帧，且之前没有发送过 Close 帧，那这个端点必须发送一个 Close 帧作为响应。但是 Close 帧的发送可能由于当前消息的处理而延迟发送(比如整个消息的大部分片段已经发送，发送完剩下的片段才会发送 Close 帧)。但是不保证发送了 Close 帧后还会继续处理数据。

一个端点在发送并接收到 Close 消息之后，端点可以认为 Websocket 已经关闭，必须马上关闭底层的 TCP。服务端必须马上关闭底层 TCP；客户端应该等服务端先关闭连接，但是也可能在发送和接收了 Close 消息之后的任何时间关闭，比如一段时间后仍然没有收到服务端的 TCP 关闭握手消息，客户端就可能会先发起 TCP 关闭握手

如果客户端和服务两端同时发送了一个 Close 消息，两端都已经发送了并且接收了一个 Close 消息，应该考虑Websocket 连接已经关闭，并且关闭底层的 TCP

```
1. ClientSender -- Close --> ServerReceiver
2. ClientSender <-- Close -- ServerReceiver (由于之前数据未发送或接收完，Close 的发送可能会延时)
3. ClientSender Wait...一段时间后如果还未进入第 4 步，ClientSender 主动发起 TCP(FIN A)
4. ClientSender <-- TCP(FIN A) -- ServerReceiver
5. ClientSender -- TCP(ACK A+1) --> ServerReceiver
6. ClientSender -- TCP(FIN B) --> ServerReceiver
7. ClientSender <-- TCP(ACK B+1) -- ServerReceiver

1. ServerSender -- Close --> ClientReceiver
2. ServerSender <-- Close -- ClientReceiver (由于之前数据未发送或接收完，Close 的发送可能会延时)
3. ServerSender -- TCP(FIN A) --> ClientReceiver
4. ServerSender <-- TCP(ACK A+1) -- ClientReceiver
    ......
    ......

第 1 和第 2 步 Server 和 Client 也可以同时发送 Close，同时响应 Close，但是最后还是 Server 先发起 TCP 关闭握手
```

#### 端点的关闭信号发送接收与数据处理
- 已经发送了 Close 帧，端点不会再发送更多数据；已经收到了 Close 帧的，端点会忽略更多数据的接收
- 收到了 Close 帧但是有些数据还在发送的，会继续发送完，然后还要发响应 Close 帧；收到了 Close 帧但是有些数据还在接收的，会继续接收完，然后还要发响应 Close 帧

这种设计避免了不必要的数据损失，也是对 TCP 关闭握手的一种完善

<a id="byte_order" />

# 字节序
- 大端字节序 big endian
    - 高位字节在前，低位字节在后，这是人类读写数值的方法
- 小端字节序 little endian
    - 低位字节在前，高位字节在后
    

## 字节序举例

0x1234567 4个字节存储

内存地址 0x100, 0x101, 0x102, 0x103

大端(第一行) 小端(第二行) 分别为

0x100|0x101|0x102|0x103
---|---|---|---
0x01|0x23|0x45|0x67
0x67|0x45|0x23|0x01



# 文献

The WebSocket Protocol: [IETF RFC 6455](#https://tools.ietf.org/html/rfc6455)