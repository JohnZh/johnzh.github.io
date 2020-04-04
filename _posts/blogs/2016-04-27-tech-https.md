---
layout: post
title:  "HTTPS，SSL/TLS，数字证书"
date:   2016-04-27 00:00:00 +0800
categories: [tech]
---
Last modified：2018-04-04

# HTTPS 

HTTPS：HyperText Transfer Protocol Secure

也可以理解为 HTTP over TLS 或者 HTTP over SSL，通过 HTTP 通信，利用 SSL/TLS 进行数据包加密，保护数据隐私与完整性。

严格说，HTTPS 不是单独协议，而是对工作在加密连接(TSL 或 SSL)上常规HTTP的称呼

**开发 HTTPS 主要目的是为了提供对网站服务器的身份认证**

# vs HTTP
- HTTP URL 由 "http://" 开始，默认端口 80；HTTPS URL 由 "https://" 开始，默认端口 443
- HTTP不是安全的，攻击者可以通过监听和中间人攻击等手段，获取网站帐户和敏感信息等；HTTPS 的设计可以防止前述攻击，在正确配置时是安全的

# TLS/SSL
- TLS：Transport Layer Security，前身 SSL：Secure Sockets Layer
- 是一种安全协议：为互联网通信提供安全及数据完整性保障
- 也是应用层协议，工作于 HTTP 之下，传输层之上

## SSL -> TLS
- SSL 分为 1.0，2.0，3.0 三个版本，1.0 和 2.0 因存在严重安全漏洞被3.0 替代，其中 1.0 未公布过
- 3.0 1996 年发布，作为历史文献 IETF 通过 RFC 6101 发表

> IETF：Internet Engineering Task Force，互联网工程任务组。是一个开放的标准组织，负责开发和推广自愿互联网标准，构成 TCP/IP 协议族（TCP/IP）的标准

- TLS 1.0，IETF 将 SSL 标准化，即 RFC 2246。技术上来说，TLS 1.0 和 SSL 3.0 差异微小，但足以排除两者之间的互操作性
- TLS 1.0 包括降级到 SSL 3.0 的实现，但这削弱了连接安全性

> 2014.10 Google 发布了在 SSL 3.0 中的设计缺陷，建议禁用，且公司相关产品中陆续禁止回溯兼容，强制使用TLS协议。Mozilla 11.25 也发布 FIREFOX 中彻底禁用 SSL 3.0。

## TLS 发展
- 2006.4，TLS 1.1，RFC 4346
- 2008.8，TLS 1.2，RFC 5246
- 2018.8，TLS 1.3，RFC 8446

版本间的区别多为密码学相关内容，例如算法的替换，新增算法支持，不展开

## TLS 基础握手

#### 1. Client -> Server
- Client 发送 `"ClientHello"` 消息到 Server

```
ClientHello:
- TLS 最高协议版本，如 TLS 1.2
- 随机数，用于生成 "会话秘钥"
- 支持的加密套件列表，如 RSA，DH，DHE
- 支持的压缩算法
```

#### 2. Server -> Client
- Server 发送 `"ServerHello"` 消息到 Client

```
ServerHello:
- 确定的协议版本 (选双方都支持的最高版本)
- 随机数
- 确定的加密套件
- 确定的压缩算法
```
- Server 发送 `"Certificate"` 消息，含 Server 证书 (含公钥 *serverPublicKey*)，该步骤可能会由于加密套件的不同而被 Server 省略
- Server 发送 `"ServerKeyExchange"` 消息，该步骤可能会由于加密套件的不同而被 Server 省略，选择 DHE 和 DH_ANON 加密套件都会发送该消息
- Server 发送 `" ServerHelloDone"` 消息

#### 3. Client -> Server
- Client 验证 Server 证书有效性

> 验证 Server 证书有效性：如果证书不是可信机构颁布，或者证书中的域名与实际域名不一致，或者证书已过期，访问者会被显示一个警告，选择是否还要继续通信

- Client 发送 `"ClientKeyExchange"` 消息，包含 PreMasterSecret ，公钥 *clientPublicKey*，或者什么也没有，取决于选中的加密套件。PreMasterSecret 会被 Server 证书里面的公钥加密过。
- Server 通过 *serverPrivateKey* 解密得 PreMasterSecret，Client 和 Server 使用之前的随机数和 PreMasterSecret 计算出 MasterSecret：MS，作为对称加密秘钥
- Client 发送 `"ChangeCipherSpec"` 记录，告诉 Server 从现在开始发送内容都会被验证(以及被加密)。

> ChangeCipherSpec 其实已经是 TLS 记录层协议内容，Content type 字段值为 20 (0x14)

- Client 发送一个被验证过且被加密过的 `"Finish"` 消息，包含了之前握手消息的 Hash 值和 Mac 值

- Server 会解密 Clinet 的 `"Finish"` 信息，并验证 Hash 值和 Mac 值。如果解密或者验证失败，连接会结束。

#### 4. Server -> Client

- Server 也会发出 `"ChangeCipherSpec"`，告诉 Client 从现在开始发送内容都会被验证(以及被加密) 
- Server 发送它的被验证过且被加密过的 `"Finish"` 消息
- Client 进行和之前 Server 同样的解密和验证。

## TLS 双向认证握手

#### 1. 与基础握手一致

#### 2. Server -> Client
- `"ServerHello"`
- `"Certificate"`
- `"ServerKeyExchange"`，该步骤可能会由于加密套件的不同而被 Server 省略，选择 DHE 和 DH_ANON 加密套件都会发送该消息
- Server 发送 `"CertificateRequest"` 消息，请求 Client 的证书做双向认证
- `" ServerHelloDone"`

#### 3. Client -> Server
- Client 验证 Server 证书有效性
- Client 发送 `"Certificate"` 消息响应 Server 的证书请求，消息包含 Client 的证书 (包含 *clientPublicKey*)
- Client 发送 `"ClientKeyExchange"` 消息，包含 PreMasterSecret，公钥 *clientPublicKey*，或者什么也没有，取决于选中的加密套件。PreMasterSecret 会被 Server 证书里面的公钥加密过。
- Client 发送 `"CertificateVerify"` 消息，这是使用 Client 证书私钥 *clientPrivateKey* 签名之前握手消息而得到的[数字签名](#digital_signature)，Server 可以用 *clientPublicKey* 验签。验签让 Server 知道 Client 能访问其证书的权力以及拥有该证书
- Server 通过 *serverPrivateKey* 解密得 PreMasterSecret，Client 和 Server 使用之前的随机数和 PreMasterSecret 计算出 MasterSecret：MS，作为对称加密秘钥
- `"ChangeCipherSpec"` 记录
- `"Finish"`

#### 4. 与基础握手一致

## 握手过程知识点小结
握手过程中使用到了非对称加密(公钥-私钥对)
1. Client 用 *serverPublicKey* 加密 PreMasterSecret，Server 用 *serverPrivateKey* 解密获得 PreMasterSecret
2. Client 用其证书的 *clientPrivateKey* 签名之前发送的消息，Server 用 *clientPublicKey* 验签
3. CA 证书有效性验证上也使用了非对称加密技术

对称加密会在之后传输数据时使用，密钥为握手过程产生的 MS。这也称为 TLS/SSL 的记录层 (**Record Layouer**)

> 这也突出了非对称加密和对称加密的特点：
> - 非对称加密加密和解密花费时间较长，速度相对较慢，适合少量数据
> - 对称加密反之，适合数据量大的情况

# 证书
证书，即数字证书，也叫身份证书，或者公开秘钥认证。是用来公开秘钥基础建设信息的电子文件，证明公开秘钥拥有者的身份。包含信息(X.509格式规范)：
- 版本
- 序号
- 主体：国家，州/省，地域/城市，组织/单位，通用名称
- 数字证书认证机构，即数字证书发行者，一般为政府或者政府授权机构
- 有效期开始时间
- 有效期结束时间
- 公钥用途
- 公钥
- 公钥指纹 (会标注出明散列算法，如 SHA1，SHA256)
- [数字签名](#digital_signature)
- 主体别名

## 证书申领
1. 申领人使用密码学产生一对密钥
2. 保留私钥，公钥连同主体信息，使用目的等信息发送给认证机构 CA
3. CA 核实身份(多渠道核实)
4. 核实信任后，补充信息，如有效期，组成证书基本信息
5. CA 用自己的私钥对申领人的公钥加上[数字签名](#digital_signature)，生成证书
6. 发送给申领人，或者公布

<a id="digital_signature" />

## 数字签名
数字签名在验证证书有效性上是一个非常重要的技术。而证书有效性的验证即身份认证。

简单的说，原文 --[散列算法，eg. sha1]--> 数字指纹 --[私钥加密：签名] --> 数字签名。验签：使用公钥解密得到数字指纹

严格的说，散列算法，也叫**数字摘要**技术，也叫 Hash， 并非数字签名必须的步骤，但是为了避免原文内容过长，加密效率低，因此通常原文会先 Hash 成短内容再签名，提高效率。

> Hash 算法特点：相同内容 Hash 出来的密文 (指纹) 必然相同，不同内容 Hash 出来的密文 (指纹) 必然不同。

综上，证书有效性的验证：除去有效期，主体等明确内容的合法性判断
- 本地利用 CA 根证书进行验签(一般 CA 根证书本地都有)，通过 CA 根证书的公钥能否解密来判断是否合法
- 解密成功，得到数字指纹，对比证书的数字指纹，对比证书的公钥进行Hash 后的数字指纹，判断是否被篡改

同理，对于握手期间 Client 签名之前发送过的内容并发送给 Server，然后 Server 的延签和 Hash 检查也是基于数字签名技术

## 证书类型
证书类型|简单说明
---|---
自签证书|用于测试目的，无任何可信赖人签名，不被广泛信任，使用时会被软件警告
根证书|获得广泛认可，通常已预先安装在各种软件上(包括操作系统，浏览器，Email 软件等)，信任链的起点，来自于公认可靠的政府机关和企业(如 Google)。有效期长，10 年以上。此外，还有企业内部网根证书，但是只用于企业内部
中介证书|认证机构给客户签发证书时候会先签发中介证书，再为客户做数字签署。有效期短，不同类别客户不同中介证书
授权证书|又叫属性证书，没有公钥，需依附于一张有效的数字证书上才有意义，赋予拥有人签发 "终端实体证书" 的权力。短期授予证书机构签发权力。
终端实体证书|不会用于签发其他证书，实际软件中部署，在创建加密通道时应用
TLS服务器证书|**主体的通用名称**为域名，**组织或单位**为机构名。证书(公钥)和私钥会安装在服务器上，等待客户端软件连接时协商加密细节。
通配符证书|"服务器证书" 上**主体的通用名称**（或主体别名）一栏以通配符为前缀，该证书可用于旗下所有子域名
TLS客户端证书|不常见，因为考虑到技术门槛及成本因素，通常都是由服务提供者验证客户身份，而不是依赖第三方认证机构。需要使用到客户端证书的一般为企业内部网，亦或是金融领域对客户安全性要求较高的，如 U 盘(盾)内置证书