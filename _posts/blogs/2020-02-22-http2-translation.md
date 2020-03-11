---
layout: post
title:  "HTTP/2"
date:   2020-02-02 00:00:00 +0800
categories: [translation]
---
# 目录
1. 介绍 (Introduction)
2. HTTP/2 协议概述 (HTTP/2 Protocol Overview)
3. [启动 HTTP/2 (Starting HTTP/2)](#start_http) 
4. HTTP 帧 (HTTP Frames)
5. [流和多路复用技术 (Streams and Multiplexing)](#streams_multiplex)
6. 帧的定义
7. 错误码
8. HTTP 消息交换
9. 额外的 HTTP
10. 安全性考虑
11. IANA 考虑
12. 相关阅读


# 摘要 (Abstract)

> This specification describes an optimized expression of the semantics of the Hypertext Transfer Protocol (HTTP), referred to as HTTP version 2 (HTTP/2). HTTP/2 enables a more efficient use of network resources and a reduced perception of latency by introducing header field compression and allowing multiple concurrent exchanges on the same connection. It also introduces unsolicited push of representations from servers to clients.

> This specification is an alternative to, but does not obsolete, the HTTP/1.1 message syntax. HTTP's existing semantics remain unchanged.

这份说明书描述了一份超文本传输协议（HTTP）语义上的优化表达，简称 HTTP version 2(HTTP/2)。HTTP/2 引入采用头部压缩，同线路多个并发交互的方式，达到了络资源高效的使用以及延时可见性的减少。同时它也引入了服务端主动发起的推送机制。

这份说明书是 HTTP/1.1 消息语法供替代的选择，而非要淘汰 HTTP/1.1 的消息语法。HTTP 现存的语义保持不变

# 1. 介绍

# 2. HTTP/2 协议概述 (Overview)

<a id="start_http" />

# 3. 启动 HTTP/2
> An HTTP/2 connection is an application-layer protocol running on top of a TCP connection ([TCP]). The client is the TCP connection initiator.

> HTTP/2 uses the same "http" and "https" URI schemes used by HTTP/1.1. HTTP/2 shares the same default port numbers: 80 for "http" URIs and 443 for "https" URIs. As a result, implementations processing requests for target resource URIs like http://example.org/foo or https://example.com/bar are required to first discover whether the upstream server (the immediate peer to which the client wishes to establish a connection) supports HTTP/2.

> The means by which support for HTTP/2 is determined is different for "http" and "https" URIs. Discovery for "http" URIs is described in Section 3.2. Discovery for "https" URIs is described in Section 3.3.

一个 HTTP/2 连接是运行在一个 TCP ([TCP](https://httpwg.org/specs/rfc7540.html#TCP)) 连接的一个应用层协议。客户端是这个 TCP 连接的发起者。

HTTP/2 使用和 HTTP/1.1相同的 URI 方案："http" 和 "https"。共享相同的默认端口：80 用于 "http" URIs，443 用于 "https" URIs. 其结果就是, 实现处理目标类似于 http://example.org/foo or https://example.com/bar 这样的 URIs 时需要先发现上流服务器(客户端希望建立一个连接的最直接的点)是否支持 HTTP/2。

确定对 HTTP/2 的是否支持的方法在 "http" 和 "https" URIs 上是不同的。"http" URIs 的发现描述于在 [Section 3.2](#)。"https" URIs 的发现描述于在 [Section 3.3](#)。

## 3.1. HTTP/2 Version Identification

> The protocol defined in this document has two identifiers.

> - The string "h2" identifies the protocol where HTTP/2 uses Transport Layer Security (TLS) [TLS12]. This identifier is used in the TLS application-layer protocol negotiation (ALPN) extension [TLS-ALPN] field and in any place where HTTP/2 over TLS is identified.<br/><br/>
The "h2" string is serialized into an ALPN protocol identifier as the two-octet sequence: 0x68, 0x32.

> - The string "h2c" identifies the protocol where HTTP/2 is run over cleartext TCP. This identifier is used in the HTTP/1.1 Upgrade header field and in any place where HTTP/2 over TCP is identified.<br/><br/>
The "h2c" string is reserved from the ALPN identifier space but describes a protocol that does not use TLS.

文档中定义的协议有两个标识符

- 字符串 "h2" 标示了 协议中使用了安全传输协议(TLS)[TLS12]的 HTTP/2 协议。这个标识符被使用在 TLS 应用层协议协商(ALPN)扩展[TLS-ALPN]字段上以及任何基于 TLS 的 HTTP/2 被定义的地方<br/><br/>"h2"字符序列化到 ALPN 标识符中表示为 2 个字节的字符串：0x68, 0x32
- 字符串 "h2c" 标示了运行在明文的 TCP 上的 HTTP/2 协议 。此标识符被使用在了 HTTP/1.1 升级报头字段以及任何基于 TCP 的 HTTP/2 被定义的地方<br/><br/>"h2c"字符串是从 ALPN 标识符空间保留下来的，但是描述了一个不使用 TLS 的协议

协商 "h2" or "h2c" 在此文档中描述意味着传输，安全性, 分帧以及消息语义。

## 3.2. Starting HTTP/2 for "http" URIs

> A client that makes a request for an "http" URI without prior knowledge about support for HTTP/2 on the next hop uses the HTTP Upgrade mechanism (Section 6.7 of [RFC7230]). The client does so by making an HTTP/1.1 request that includes an Upgrade header field with the "h2c" token. Such an HTTP/1.1 request MUST include exactly one HTTP2-Settings (Section 3.2.1) header field.

在不预先知道是否支持 HTTP/2 的情况下，客户端发送情况跳转支持使用的是 HTTP 升级机制（Section 6.7 of [RFC7230])。客户端通过发送一个带有 "h2c" 令牌的升级报头字段的 HTTP/1.1 的请求来做这件事。这样的一个 HTTP/1.1 请求必须正确地包含一个 HTTP2-Settings ([Section 3.2.1](#)) 报头字段

```
For example:

GET / HTTP/1.1
Host: server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
> Requests that contain a payload body MUST be sent in their entirety before the client can send HTTP/2 frames. This means that a large request can block the use of the connection until it is completely sent.

> If concurrency of an initial request with subsequent requests is important, an OPTIONS request can be used to perform the upgrade to HTTP/2, at the cost of an additional round trip.

> A server that does not support HTTP/2 can respond to the request as though the Upgrade header field were absent:

在客户端能发送 HTTP/2 帧之前，包含装载体的请求必须被全文完整发送。这意味着一个巨大的请求会阻塞连接的使用，直到这个请求被完整的发送。

如果一个请求的并发后续请求是重要的，那么可以使用一个 OPTIONS 请求来执行升级到HTTP/2的操作，这样会消耗一个额外的往返成本(RTT)。

不支持 HTTP/2 的服务端可以通过好像一个没有升级报头字段的响应来响应这个请求：

```
HTTP/1.1 200 OK
Content-Length: 243
Content-Type: text/html

...
```
> A server MUST ignore an "h2" token in an Upgrade header field. Presence of a token with "h2" implies HTTP/2 over TLS, which is instead negotiated as described in Section 3.3.

> A server that supports HTTP/2 accepts the upgrade with a 101 (Switching Protocols) response. After the empty line that terminates the 101 response, the server can begin sending HTTP/2 frames. These frames MUST include a response to the request that initiated the upgrade.

服务端必须忽略升级报头字段中的 "h2" 令牌，带有 "h2" 令牌的出现意味着是基于 TLS 的 HTTP/2，它的协商会在 [Section 3.3](#) 描述

支持 HTTP/2 的服务端会使用 101 (交换协议) 的响应来接受升级。在结束 101 响应的空行后，服务端就可以开始发送 HTTP/2 帧。这些帧必须包含一个对客户端升级请求的响应，用于初始化升级。

```
For example:

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c

[ HTTP/2 connection ...
```

> The first HTTP/2 frame sent by the server MUST be a server connection preface (Section 3.5) consisting of a SETTINGS frame (Section 6.5). Upon receiving the 101 response, the client MUST send a connection preface (Section 3.5), which includes a SETTINGS frame.

> The HTTP/1.1 request that is sent prior to upgrade is assigned a stream identifier of 1 (see Section 5.1.1) with default priority values (Section 5.3.5). Stream 1 is implicitly "half-closed" from the client toward the server (see Section 5.1), since the request is completed as an HTTP/1.1 request. After commencing the HTTP/2 connection, stream 1 is used for the response.

服务端发送的第一个 HTTP/2 帧必须是一个由 SETTINGS 帧组成的连接序言 ([Section 3.5](#))。接收到 101 响应时，客户端必须发送一个连接序言 ([Section 3.5](#))，这个连接序言包含一个 SETTINGS 帧

这个之前被用于升级的 HTTP/1.1 请求会被分配一个 1 的流标识符 (见 [Section 5.1.1](#)，以及一个默认优先级 ([Section 5.3.5](#))。流 1 是一个客户端到服务端隐式的 "half-closed" 状态的流 (见 [Section 5.1](#))，因为请求是以 HTTP/1.1 请求形式完成的。在开始 HTTP/2 连接后，流 1 会被用于响应请求。

### 3.2.1. HTTP2-Settings 报头字段
> A request that upgrades from HTTP/1.1 to HTTP/2 MUST include exactly one HTTP2-Settings header field. The HTTP2-Settings header field is a connection-specific header field that includes parameters that govern the HTTP/2 connection, provided in anticipation of the server accepting the request to upgrade.

一个用于将 HTTP/1.1 升级成 HTTP/2 的请求必须包含一个 HTTP2-Settings 报头字段。这个 HTTP2-Settings 报头字段是一个特定的用于连接的字段，该字段包含了用于管理 HTTP/2 连接的参数，这些参数会被提供给预期中会接受请求升级的服务器。

HTTP2-Settings    = token68

> A server MUST NOT upgrade the connection to HTTP/2 if this header field is not present or if more than one is present. A server MUST NOT send this header field.

如果这个报头字段没有出现，胡总出现多于一个，那么这个服务器将无法升级到 HTTP/2 的连接。服务端不能发送这个报头字段。

> The content of the HTTP2-Settings header field is the payload of a SETTINGS frame (Section 6.5), encoded as a base64url string (that is, the URL- and filename-safe Base64 encoding described in Section 5 of [RFC4648], with any trailing '=' characters omitted). The ABNF [RFC5234] production for token68 is defined in Section 2.1 of [RFC7235].

> Since the upgrade is only intended to apply to the immediate connection, a client sending the HTTP2-Settings header field MUST also send HTTP2-Settings as a connection option in the Connection header field to prevent it from being forwarded (see Section 6.1 of [RFC7230]).

> A server decodes and interprets these values as it would any other SETTINGS frame. Explicit acknowledgement of these settings (Section 6.5.3) is not necessary, since a 101 response serves as implicit acknowledgement. Providing these values in the upgrade request gives a client an opportunity to provide parameters prior to receiving any frames from the server.

这个 HTTP2-Settings 报头字段的内容是一个 [SETTINGS](#) 帧 ([Section 6.5](#)) 的负载，并且编码成了一个 base64url 字符串 (即 Section 5 of [RFC4648] 里面描述的 URL 和文件名安全的 Base64 编码，且是忽略掉任何后续的"="字符)。[ABNF] [RFC5234] 产品对 token68 的定义在 Section 2.1 of [RFC7235] 中。

由于升级只适用于及即时连接，因此客户端发送的 HTTP-Settings 报头字段也会作为 Connection 报头字段的选项发送，从而避免转发 (见 Section 6.1 of [RFC7230])。

服务器就像对其他 SETTINGS 帧一样对这些值进行解码和解释。因为服务端有一个 101 响应会作为一个隐式确认，对这些设置的明确确认 ([Section 6.5.3](#)) 是不必要的。在升级请求中提供这些值也是给客户端一个在接收服务端任何帧之前可以提供参数的机会。

## 3.3. Starting HTTP/2 for "https" URIs
> A client that makes a request to an "https" URI uses TLS [TLS12] with the application-layer protocol negotiation (ALPN) extension [TLS-ALPN].

> HTTP/2 over TLS uses the "h2" protocol identifier. The "h2c" protocol identifier MUST NOT be sent by a client or selected by a server; the "h2c" protocol identifier describes a protocol that does not use TLS.

> Once TLS negotiation is complete, both the client and the server MUST send a connection preface (Section 3.5).

客户端使用 TLS[TLS12] 应用层协议协商扩展[TLS-ALPN] (在不知道服务端是否支持 HTTP/2 的情况下)。

基于 TLS 的 HTTP/2 会使用 "h2" 协议标识符。"h2c" 协议标识符不能被客户端发送或者被服务端选择。"h2c" 协议标识符描述了一个不使用 TLS 的协议。

一旦 TLS 协商完成，客户端和服务端都必须发送连接序言 ([Section 3.5](#))。

## 3.4. Starting HTTP/2 with Prior Knowledge
> A client can learn that a particular server supports HTTP/2 by other means. For example, [ALT-SVC] describes a mechanism for advertising this capability.

> A client MUST send the connection preface (Section 3.5) and then MAY immediately send HTTP/2 frames to such a server; servers can identify these connections by the presence of the connection preface. This only affects the establishment of HTTP/2 connections over cleartext TCP; implementations that support HTTP/2 over TLS MUST use protocol negotiation in TLS [TLS-ALPN].

> Likewise, the server MUST send a connection preface (Section 3.5).

> Without additional information, prior support for HTTP/2 is not a strong signal that a given server will support HTTP/2 for future connections. For example, it is possible for server configurations to change, for configurations to differ between instances in clustered servers, or for network conditions to change.

客户端可以通过其他方式来判断一个服务器是否支持 HTTP/2。例如 [ALT-SVC] 描述了一种使用广告的机制。

客户端肯定会发送连接序言 ([Section 3.5](#))，然后也许还会马上发送 HTTP/2 帧到服务端；服务端能通过出现的连接序言快速的区分这些连接。这只会影响基于明文 TCP 的 HTTP/2 连接的建立；实现基于 TLS 的 HTTP/2 的支持必须使用TLS协议协商[TLS-ALPN]

同样，服务端也必须发送一个连接序言 ([Section 3.5](#))。

在没有额外信息的情况下，先前支持 HTTP/2 的服务器并不能表示在未来的连接上依旧支持 HTTP/2。例如服务端可能改变配置，集群服务器可能存在个体差异，网络条件可能发生变化。

## 3.5. HTTP/2 连接序言 (Preface)
> In HTTP/2, each endpoint is required to send a connection preface as a final confirmation of the protocol in use and to establish the initial settings for the HTTP/2 connection. The client and server each send a different connection preface.

> The client connection preface starts with a sequence of 24 octets, which in hex notation is:

> 0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a

> That is, the connection preface starts with the string PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n). This sequence MUST be followed by a SETTINGS frame (Section 6.5), which MAY be empty. The client sends the client connection preface immediately upon receipt of a 101 (Switching Protocols) response (indicating a successful upgrade) or as the first application data octets of a TLS connection. If starting an HTTP/2 connection with prior knowledge of server support for the protocol, the client connection preface is sent upon connection establishment.

> **Note:** The client connection preface is selected so that a large proportion of HTTP/1.1 or HTTP/1.0 servers and intermediaries do not attempt to process further frames. Note that this does not address the concerns raised in [TALKING].

在 HTTP/2 中，每端都需要发送连接序言作为确认以及为 HTTP/2 连接建立初始设置。客户端和服务器发送的连接序言是不同的。

客户端的连接序言以一个 24 字节序列开始，其十六进制表示如下：

0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a

即，连接序言以字符串 PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n 开始。序列后必须跟着一个 SETTINGS 帧 ([Section 6.5](#))，此帧可以是空帧。客户端在收到 101 (交换协议) 响应后马上发送客户端连接序言，或者作为TLS 连接的第一个应用数据。如果先前已经知道服务端支持该协议，客户端连接序言会在建立连接时候发送。

**注意：**客户端连接序言是为了让大部分 HTTP/1.1 或者 HTTP/1.0 服务端以及中介端不尝试处理更多帧而使用的。注意这并没有解决掉 [TALKING] 里面提及问题。

> The server connection preface consists of a potentially empty SETTINGS frame (Section 6.5) that MUST be the first frame the server sends in the HTTP/2 connection.

> The SETTINGS frames received from a peer as part of the connection preface MUST be acknowledged (see Section 6.5.3) after sending the connection preface.

> To avoid unnecessary latency, clients are permitted to send additional frames to the server immediately after sending the client connection preface, without waiting to receive the server connection preface. It is important to note, however, that the server connection preface SETTINGS frame might include parameters that necessarily alter how a client is expected to communicate with the server. Upon receiving the SETTINGS frame, the client is expected to honor any parameters established. In some configurations, it is possible for the server to transmit SETTINGS before the client sends additional frames, providing an opportunity to avoid this issue.

> Clients and servers MUST treat an invalid connection preface as a connection error (Section 5.4.1) of type PROTOCOL_ERROR. A GOAWAY frame (Section 6.8) MAY be omitted in this case, since an invalid preface indicates that the peer is not using HTTP/2.

服务端连接序言由一个可以为空工的 SETTINGS 帧组成，且必须作为服务端 HTTP/2 连接的首帧发送。

接收到来自另一端的 SETTINGS 帧，且为连接序言的一部分的，在自己发送了连接需要后，必须被确认 (见 [Section 6.5.3](#)) 

为了避免不必要的延时，客户端允许在发送连接序言后马上发送其他帧到服务端，且不需要等待接收到服务器的连接序言之后。不过要注意的是，服务端连接序言的 SETTINGS 帧也许包含了一些希望客户端如何和服务端交流的参数。在收到服务端的 SETTINGS 帧后，客户端应准守这些设置参数。在有一些配置中，有可能服务端会在客户端发送其他帧之前传送 SETTINGS 帧，目的就是为了避免错误。

客户端和服务端必须把无效的连接序言作为一个 PROTOCOL_ERROR 连接错误 ([Section 5.4.1](#))。在这种情况下，GOAWAY 帧 ([Section 6.8](#)) 可能会被省略，因为无效序言表明对端并没有使用 HTTP/2。

# 4. HTTP 帧


<a id="streams_multiplex" />

# 5. 流和多路复用技术
> A "stream" is an independent, bidirectional sequence of frames exchanged between the client and server within an HTTP/2 connection. Streams have several important characteristics:
> - A single HTTP/2 connection can contain multiple concurrently open streams, with either endpoint interleaving frames from multiple streams.
> - Streams can be established and used unilaterally or shared by either the client or server.
> - Streams can be closed by either endpoint.
The order in which frames are sent on a stream is significant. Recipients process frames in the order they are received. In particular, the order of HEADERS and DATA frames is semantically significant.
> - Streams are identified by an integer. Stream identifiers are assigned to streams by the endpoint initiating the stream.

流是在客户端和服务端的 HTTP/2 连接中用作数据交换的独立的双向的帧序列。其有如下特点：

- 一个单独的 HTTP/2 连接能包含多个并发打开的流，且任意端点的交错的帧来自于多个流
- 流可以由客户端或者服务端单方面建立使用或分享
- 流能被任意端点关闭。流真帧的顺序是非常重要的。接收者按帧被接收的顺序处理。特别是报头帧和数据帧的顺序语义上十分重要。
- 流通过一个整数标识。流标识符会在端点初始化流的时候分配到流上。

## 5.1. 流的状态
流的生命周期，如图 2：
```
                             +--------+
                     send PP |        | recv PP
                    ,--------|  idle  |--------.
                   /         |        |         \
                  v          +--------+          v
           +----------+          |           +----------+
           |          |          | send H /  |          |
    ,------| reserved |          | recv H    | reserved |------.
    |      | (local)  |          |           | (remote) |      |
    |      +----------+          v           +----------+      |
    |          |             +--------+             |          |
    |          |     recv ES |        | send ES     |          |
    |   send H |     ,-------|  open  |-------.     | recv H   |
    |          |    /        |        |        \    |          |
    |          v   v         +--------+         v   v          |
    |      +----------+          |           +----------+      |
    |      |   half   |          |           |   half   |      |
    |      |  closed  |          | send R /  |  closed  |      |
    |      | (remote) |          | recv R    | (local)  |      |
    |      +----------+          |           +----------+      |
    |           |                |                 |           |
    |           | send ES /      |       recv ES / |           |
    |           | send R /       v        send R / |           |
    |           | recv R     +--------+   recv R   |           |
    | send R /  `----------->|        |<-----------'  send R / |
    | recv R                 | closed |               recv R   |
    `----------------------->|        |<----------------------'
                             +--------+

       send:   endpoint sends this frame
       recv:   endpoint receives this frame

       H:  HEADERS frame (with implied CONTINUATIONs)
       PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
       ES: END_STREAM flag
       R:  RST_STREAM frame
```

> Figure 2: Stream States

> Note that this diagram shows stream state transitions and the frames and flags that affect those transitions only. In this regard, CONTINUATION frames do not result in state transitions; they are effectively part of the HEADERS or PUSH_PROMISE that they follow. For the purpose of state transitions, the END_STREAM flag is processed as a separate event to the frame that bears it; a HEADERS frame with the END_STREAM flag set can cause two state transitions.

> Both endpoints have a subjective view of the state of a stream that could be different when frames are in transit. Endpoints do not coordinate the creation of streams; they are created unilaterally by either endpoint. The negative consequences of a mismatch in states are limited to the "closed" state after sending RST_STREAM, where frames might be received for some time after closing.

> Streams have the following states: 

图 2：流状态

注意该图只展示了流状态的转换以及影响这些转换的帧和标志。在这方面，CONTINUATION 帧不会导致状态转换；CONTINUATION 帧跟在 HEADERS 或者 PUSH_PROMOISE 帧的后面，属于他们的一部分。为了达到状态转换的目的，支持流状态转化的帧，其 END_STREAM 标志会被作为一个单独事件来处理。 标志会被当作单独事件来处理，用于支持流状态的转换；设有 END_STREAM 标志的 HEADERS 帧会造成两个状态转换。

帧在传输的过程中，各端在流的状态的主观认知上可能是不同的。终端不会协调流的创建；流会被任何一端单方面创建。状态不匹配的消极影响是"closed"的状态受限制，"closed" 的状态发生在发送 RST_STREAM 帧 后，帧可能会在流关闭后的一段时间后背收到

> All streams start in the "idle" state.

> The following transitions are valid from this state:
> - Sending or receiving a HEADERS frame causes the stream to become "open". The stream identifier is selected as described in Section 5.1.1. The same HEADERS frame can also cause a stream to immediately become "half-closed".
> - Sending a PUSH_PROMISE frame on another stream reserves the idle stream that is identified for later use. The stream state for the reserved stream transitions to "reserved (local)".
> - Receiving a PUSH_PROMISE frame on another stream reserves an idle stream that is identified for later use. The stream state for the reserved stream transitions to "reserved (remote)".
> - Note that the PUSH_PROMISE frame is not sent on the idle stream but references the newly reserved stream in the Promised Stream ID field.

> Receiving any frame other than HEADERS or PRIORITY on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.

所有的流都从"空闲"状态开始：

下面的这些从"空闲"状态开始的转变是合法的：

- 发送或者接受一个 HEADERS 帧会造成流变成"打开"状态。流标示描述在[Section 5.1.1](#)。相同的 HEADERS 帧也能造成流马上变成"半关闭"状态。
- 在另一个流上发送一个 PUHS_PROMISE 帧同时保留这个空闲的被标示过的用于后续使用的流。这个被保留的流的状态会转变为"保留(local)"。
- 在另一个流上接收到一个 PUSH_PROMISE 帧同时保留这个空闲的被标示过的用于后续使用的流。这个被保留的流的状态会转变为"保留(remote)"。
- 要注意的是 PUHS_PROMOSE 帧不会在空闲流上发送，但是在 Promised Stream ID 字段上会参考这个最新保留的流。

流在空闲状态下接收到任何不是 HEADERS 或者 PRIORITY 的帧必须被当成 PROTOCOL_ERROR 类型的连接错误([Section 5.4.1](#))来处理

> reserved (local):

> A stream in the "reserved (local)" state is one that has been promised by sending a PUSH_PROMISE frame. A PUSH_PROMISE frame reserves an idle stream by associating the stream with an open stream that was initiated by the remote peer (see Section 8.2).

> In this state, only the following transitions are possible:

> The endpoint can send a HEADERS frame. This causes the stream to open in a "half-closed (remote)" state.
Either endpoint can send a RST_STREAM frame to cause the stream to become "closed". This releases the stream reservation.
An endpoint MUST NOT send any type of frame other than HEADERS, RST_STREAM, or PRIORITY in this state.

> A PRIORITY or WINDOW_UPDATE frame MAY be received in this state. Receiving any type of frame other than RST_STREAM, PRIORITY, or WINDOW_UPDATE on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.

保留(local)：

在保留(local) 状态下的流是一个承诺用于发送 PUSH_PROMISE 帧的流。PUSH_PROMISE 帧通过关联流到一个由远端初始化并且是打开状态的的流上来保留一个空闲流(见 [Section 8.2](#))

在这个状态下，只有下列转变是可能的：

端点可以发送 HEADERS 帧。这会造成流打开到一个"半关闭(remote)"状态。任何一个端点都可以发送 RST_STREAM 帧来使得这个流变成"关闭"状态。这是释放流的保留。一个端点不能在这个状态下发送任何不是 HEADERS，RST_STREAM，或者是 PRIORITY 的帧。

在这个状态下，可能会收到 PRIORITY 或者 WINDOW_UPDATE 帧。流在这个状态下收到任何不是 RST_STREAM，PRIORITY，或者 WINDOW_UPDATE 类型的帧都必须被当做是 PROTOCOL_ERROR 类型的连接错误([Secion 5.4.1](#))。

> reserved (remote):

> A stream in the "reserved (remote)" state has been reserved by a remote peer.

> In this state, only the following transitions are possible:

> Receiving a HEADERS frame causes the stream to transition to "half-closed (local)".
Either endpoint can send a RST_STREAM frame to cause the stream to become "closed". This releases the stream reservation.
An endpoint MAY send a PRIORITY frame in this state to reprioritize the reserved stream. An endpoint MUST NOT send any type of frame other than RST_STREAM, WINDOW_UPDATE, or PRIORITY in this state.

> Receiving any type of frame other than HEADERS, RST_STREAM, or PRIORITY on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.

保留(remote)：

在保留(remote)状态下的流是被远端保留下来的。

这个状态下，只有下面的转变是可能的：

接收到 HEADERS 帧会造成流转变为"半关闭(local)"状态。任一端点都可以发送 RST_STREAM 帧来使得这个流变成"关闭"状态。这是释放流的保留。在这个状态下端点可能会发生一个 PRIORITY 帧来变更这个保留的流的优先级。端点不能在这个状态下发送任何不是 HEADERS，RST_STREAM，或者是 PRIORITY 的帧。

流在这个状态下收到任何不是 HEADERS，RST_STREAM，或者 PRIORITY 类型的帧都必须被当做是 PROTOCOL_ERROR 类型的连接错误([Secion 5.4.1](#))。

> open:

> A stream in the "open" state may be used by both peers to send frames of any type. In this state, sending peers observe advertised stream-level flow-control limits (Section 5.2).

> From this state, either endpoint can send a frame with an END_STREAM flag set, which causes the stream to transition into one of the "half-closed" states. An endpoint sending an END_STREAM flag causes the stream state to become "half-closed (local)"; an endpoint receiving an END_STREAM flag causes the stream state to become "half-closed (remote)".

> Either endpoint can send a RST_STREAM frame from this state, causing it to transition immediately to "closed".

在"打开"状态下的流可以被所有的端点使用，并且发送任何类型的帧。在这种状态下，发送数据的端点会准守流级别流量控制的限制([Section 5.2](#))。

自这个状态下，任何端点发送设有 END_STREAM 标志的帧，都会造成流转变成一个"半关闭"状态。一个端点发送一个  END_STREAM 标志会导致流状态变成"半关闭(local)"状态；而另一个接收到到 END_STREAM 标志的会导致流状态变成"半关闭(remote)"状态。

自这个状态下，任一端点都能发送 RST_STREAM 帧都会造成状态马上转变为"关闭"状态。

> half-closed (local):

> A stream that is in the "half-closed (local)" state cannot be used for sending frames other than WINDOW_UPDATE, PRIORITY, and RST_STREAM.

> A stream transitions from this state to "closed" when a frame that contains an END_STREAM flag is received or when either peer sends a RST_STREAM frame.

> An endpoint can receive any type of frame in this state. Providing flow-control credit using WINDOW_UPDATE frames is necessary to continue receiving flow-controlled frames. In this state, a receiver can ignore WINDOW_UPDATE frames, which might arrive for a short period after a frame bearing the END_STREAM flag is sent.

> PRIORITY frames received in this state are used to reprioritize streams that depend on the identified stream.

半关闭(local)：

流在半关闭(local)下不能用于发送除了 WINDOW_UPDATE，PRIORITY，RST_STREAM 以外的帧

当流收到一个标志为 END_STREAM 的帧或者任一端点发生了 RST_STREAM 帧，流会从这个状态转变为"关闭"状态。

在这个状态下，端点能接收到任何类型的帧。为了持续接收流量控制帧，使用 WINDOW_UPDATE 帧来提供流量控制信用是必要的。在这个状态下，接收者能忽略 WINDOW_UPDATE 帧，而此类帧也许会在一个承载了 END_STREAM 标志的帧发出后的短期内到达。

在这个状态下，收到的 PRIORITY 帧会被用于变更流的优先级，而这些流依赖于这个被标识的流

> half-closed (remote):

> A stream that is "half-closed (remote)" is no longer being used by the peer to send frames. In this state, an endpoint is no longer obligated to maintain a receiver flow-control window.

> If an endpoint receives additional frames, other than WINDOW_UPDATE, PRIORITY, or RST_STREAM, for a stream that is in this state, it MUST respond with a stream error (Section 5.4.2) of type STREAM_CLOSED.

> A stream that is "half-closed (remote)" can be used by the endpoint to send frames of any type. In this state, the endpoint continues to observe advertised stream-level flow-control limits (Section 5.2).

> A stream can transition from this state to "closed" by sending a frame that contains an END_STREAM flag or when either peer sends a RST_STREAM frame.

半关闭(remote)：

状态是"半关闭(remote)" 的流将不会再被用于发送帧。在这个状态下，端点将不再有义务维护啊一个接受者的流量控制窗口。

如果端点接收格外的帧，而不是 WINDOW_UPDATE，PRIORITY，或者 RST_STREAM，对于为此状态下的流来说，必须响应成类型为 STREAM_CLOSED 的流错误([Section 5.4.2](#))

处于"半关闭(remote)"状态下的流可以被端点用于发送任何类型的帧。这种状态下，端点应该继续准守流级别流量控制限制([Section 5.2](#))。

通过发送包含 END_STREAM 标志的帧或者任一端点发送 RST_STREAM 帧，流可以从这个状态转变为"关闭"状态。

> closed:

> The "closed" state is the terminal state.

> An endpoint MUST NOT send frames other than PRIORITY on a closed stream. An endpoint that receives any frame other than PRIORITY after receiving a RST_STREAM MUST treat that as a stream error (Section 5.4.2) of type STREAM_CLOSED. Similarly, an endpoint that receives any frames after receiving a frame with the END_STREAM flag set MUST treat that as a connection error (Section 5.4.1) of type STREAM_CLOSED, unless the frame is permitted as described below.

> WINDOW_UPDATE or RST_STREAM frames can be received in this state for a short period after a DATA or HEADERS frame containing an END_STREAM flag is sent. Until the remote peer receives and processes RST_STREAM or the frame bearing the END_STREAM flag, it might send frames of these types. Endpoints MUST ignore WINDOW_UPDATE or RST_STREAM frames received in this state, though endpoints MAY choose to treat frames that arrive a significant time after sending END_STREAM as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.

> PRIORITY frames can be sent on closed streams to prioritize streams that are dependent on the closed stream. Endpoints SHOULD process PRIORITY frames, though they can be ignored if the stream has been removed from the dependency tree (see Section 5.3.4).

> If this state is reached as a result of sending a RST_STREAM frame, the peer that receives the RST_STREAM might have already sent — or enqueued for sending — frames on the stream that cannot be withdrawn. An endpoint MUST ignore frames that it receives on closed streams after it has sent a RST_STREAM frame. An endpoint MAY choose to limit the period over which it ignores frames and treat frames that arrive after this time as being in error.

> Flow-controlled frames (i.e., DATA) received after sending RST_STREAM are counted toward the connection flow-control window. Even though these frames might be ignored, because they are sent before the sender receives the RST_STREAM, the sender will consider the frames to count against the flow-control window.

> An endpoint might receive a PUSH_PROMISE frame after it sends RST_STREAM. PUSH_PROMISE causes a stream to become "reserved" even if the associated stream has been reset. Therefore, a RST_STREAM is needed to close an unwanted promised stream.

关闭：

"关闭" 状态是终结状态。

端点在关闭状态的流上不能发送除 PRIORITY 以外的帧。端点在收到 RST_STREAM 帧后如果依然接收到了除 PRIORITY 以外的帧，则必须作为 STREAM_CLOSED 类型的流错误对待([Section 5.4.2](#))。同样的，端点在收到带有 END_STREAM 标志的帧之后再收到任何帧，则则必须作为 STREAM_CLOSED 类型的连接错误对待([Section 5.4.1](#))，除非如下描述的这些被允许的帧。

在这个状态下，端点发送在包含 END_STREAM 标志 DATA 和 HEADERS 帧发送后的一小短时间内，都可以接收 WINDOW_UPDATE 或者 RST_STREAM 帧。直到远端端点接收并处理完 RST_STREAM 或者承载着 END_STREAM 标志的帧之前，端点都可以发送这些类型的帧。尽管端点也许会把那些在发送了 END_STREAM 之后一段时间内到达的帧当做 PROTOCOL_ERROR 类型连接错误([Section 5.4.1](#))，但是在这个状态下端点肯定会忽略 收到的 WINDOW_UPDATE 或者 RST_STREAM 帧。

PRIORITY 帧可以在关闭的流上发送用来给依赖于这个关闭流的其他流设置优先级。端点应该处理 PRIORITY 帧，尽管在流已经从依赖树(见 [Section 5.3.4](#))移除的情况下 PRIORITY 帧可以被忽略。

如果已经由于发送了 RST_STREAM 而变成这个状态，那么接收了 RST_STREAM 的端点已经发送的，或者正在发送的，都在流上了的这些帧都是不能退出的。发送了 RST_STREAM 的端点必须忽略那些关闭的流上的帧。端点可以选择设置超时时间，超过超时时间后会忽略帧并把它们作为错误处理。

在发送了 RST_STREAM 后收到的流量控制帧(例如 DATA)都被会计入连接流量控制窗口。尽管这些帧可以被被忽略，因为他们是被发送于发送者接收到 RST_STREAM 之前，但是发送者会认为这些帧不利于流量控制窗口。

端点在发送了 RST_STREAM 后也许会收到 PUSH_PROMISE 帧。PUSH_PROMISE 会造成流变成 "保留" 状态，即便关联的流已经被重置。因此，RST_STREAM 用于关闭一个不需要的流是需要的。

> In the absence of more specific guidance elsewhere in this document, implementations SHOULD treat the receipt of a frame that is not expressly permitted in the description of a state as a connection error (Section 5.4.1) of type PROTOCOL_ERROR. Note that PRIORITY can be sent and received in any stream state. Frames of unknown types are ignored.

> An example of the state transitions for an HTTP request/response exchange can be found in Section 8.1. An example of the state transitions for server push can be found in Sections 8.2.1 and 8.2.2.

文档中缺乏特殊说明的，某个状态的描述中关于帧的接收没有明确允许的，这些实现方式应该采用作为 PROTOCOL_ERROR 类型的连接错误([Section 5.4.1](#))来处理。注意，PRIORITY 帧可以在任何流状态下被发送或者接收。未知的帧类型应该被忽略。

HTTP 请求/响应过程中状态转变的例子可以在 Section 8.1 里面找到。服务器推送的状态转变的例子可以在 Sections 8.2.1 和 8.2.2 里面找到。

### 5.1.1. 流标识符

## 5.2. 流量控制

## 5.3. 流的优先级

## 5.4. 错误处理

## 5.5. Extending HTTP/2

# 6. 帧的定义

# 7. 错误码

# 8. HTTP 消息交换

# 9. 额外的 HTTP

# 10. 安全性考虑

# 11. IANA 考虑

# 12. 相关阅读