# HTTP/3

## HTTP 1

- HTTP/1.0 传输数据时，每次都需要重新建立连接，增加延迟。
- HTTP/1.1 虽然加入 keep-alive 可以复用一部分连接，但域名分片等情况下仍然需要建立多个 connection，耗费资源，给服务器带来性能压力
- Head-Of-Line Blocking（HOLB） 队头阻塞
- header 里携带的内容过大


## HTTP 2
-  二进制传输
- HTTP/2 将请求和响应数据分割为更小的帧，并且它们采用二进制编码。
- 多路复用，  解决了浏览器限制同一个域名下的请求数量的问题     一个连接并行发送多个请求和响应
- Header 压缩，客户端和服务器端使用“首部表”来跟踪和存储之前发送的键－值对，对于相同的数据，不再通过每次请求和响应发送
- Server Push

- 缺点: 丢包   整个 TCP 都要开始等待重传

## HTTP 3
- QUIC on top of UDP
- QUIC 是基于 UDP 的，一个连接上的多个 stream 之间没有依赖。比如下图中 stream2 丢了一个 UDP 包，不会影响后面跟着 Stream3 和 Stream4，不存在 TCP 队头阻塞
- UDP 缺乏的可靠性由QUIC提供，数据包重传，拥塞控制
- HTTPS 的一次完全握手的建连过程，需要 3 个 RTT。 HTTP 3 建立在UDP 的基础上，实现了 0RTT
- 传输层的优化，QUIC on top of UDP on user space，不需要操作系统升级
- HTTP/3 在协议层相比HTTP/2也是不同的，一些HTTP/2 的功能在QUIC层面已经实现 (stream flow control)，所以HTTP/3会去掉重复的功能
- 数据流有connection ID，单一数据流，可靠，有序；不同数据流间无序传送。 在IP和网络迁移情况下仍然保持连接（connection ID没有变），如5G转换到WIFI，这点TCP做不到。

- 缺点

    * linux 内核对tcp堆栈有优化
    * TCP TLS有硬件加速
    * 网络中间层节点对UDP不友好，拦截，节流



&nbsp;

## RTT

&nbsp;

### TCP连接

 * 一去 （SYN）

 * 二回 （SYN+ACK）

 * 三去 （ACK）

 * 相当于一个半来回，故TCP连接的时间 = 1.5 RTT 。

### HTTP 数据

 * 一去（HTTP Request）

 * 二回 （HTTP Responses）

 * 故HTTP的交易时间 = 1 RTT

 * TCP + HTTP = 2.5 RTT，从TCP开始握手到得到第一个Http response 要2.5 RTT

### TLS

 * Client Hello
 * Server Hello
 * Key Exchange
 * 1.5 RTT
 * HTTPS = TCP + TLS + HTTP = 1.5 + 1.5 + 1 = 4 RTT

<img src="img/http-request-over-tcp-tls.png?raw=true"  width="900">

### HTTP 3

<img src="img/http-request-over-quic.png?raw=true"  width="900">


<img src="img/http3.png?raw=true"  width="900">

```
Client                                        Server
----> CHLO
                       <---- REJ(
				Server config：
				Server长期公钥
				CERT CHAIN
				用Server私钥的签名    (验证服务器是真正的)
				source address token（服务端认证client用）
				)
----> COMPLETE CHLO（client短期公钥）
此时client可以算出init keys(client私钥， 服务端长期公钥)
用init keys来加密第一份http data，随着COMPLETE CHLO一起送到Server

		<----   SHLO(用init keys加密) 包含一份Server的短期公钥
		forward secure keys（client 短期公钥， 服务端短期公钥）
		用forward secure keys加密回复client
		
---->  client收到Server的短期公钥 改用forward secure keys加密data

这里的秘钥有2种，第一种是client的第一次加密数据用的init key，后面的数据交互都是用的forward secure keys

以上的连接花了2个RTT来送出第一份request/response数据。
如果client缓存了server config，那么他可以直接发起 COMPLETE CHLO， 相当于立马送出request数据，也就是网上说的0-RTT。
当source address token或者Server config失效的时候，才会再次服务端送出REJ。1-RTT handshake
```


## [原文资料](https://dl.acm.org/doi/pdf/10.1145/3098822.3098842)

&nbsp;
<img src="img/quic.png?raw=true"  width="900">

&nbsp;
```
QUIC relies on a combined cryptographic and transport handshake for setting up a secure transport connection. On a successful handshake, a client caches information about the origin3 . On subsequent connections to the same origin, the client can establish an encrypted connection with no additional round trips and data can be sent immediately following the client handshake packet without waiting for a reply from the server. QUIC provides a dedicated reliable stream (streams are described below) for performing the cryptographic handshake. This section summarizes the mechanics of QUIC’s cryptographic handshake and how it facilitates a zero roundtrip time (0-RTT) connection setup. Figure 4 shows a schematic of the handshake. 

Initial handshake: Initially, the client has no information about the server and so, before a handshake can be attempted, the client sends an inchoate client hello (CHLO) message to the server to elicit a reject (REJ) message. 

The REJ message contains: 
(i) a server config 3An origin is identified by the set of URI scheme, hostname, and port number [5]. that includes the server’s long-term Diffie-Hellman public value, 
(ii) a certificate chain authenticating the server, 
(iii) a signature of the server config using the private key from the leaf certificate of the chain, and 
(v) a source-address token: an authenticated-encryption block that contains the client’s publicly visible IP address (as seen at the server) and a timestamp by the server. The client sends this token back to the server in later handshakes, demonstrating ownership of its IP address. 

Once the client has received a server config, it authenticates the config by verifying the certificate chain and signature. It then sends a complete CHLO, containing the client’s ephemeral Diffie-Hellman public value. 

Final (and repeat) handshake: All keys for a connection are established using Diffie-Hellman. After sending a complete CHLO, the client is in possession of initial keys for the connection since it can calculate the shared value from the server’s long-term DiffieHellman public value and its own ephemeral Diffie-Hellman private key. At this point, the client is free to start sending application data to the server. Indeed, if it wishes to achieve 0-RTT latency for data, then it must start sending data encrypted with its initial keys before waiting for the server’s reply. 

If the handshake is successful, the server returns a server hello (SHLO) message. This message is encrypted using the initial keys, and contains the server’s ephemeral Diffie-Hellman public value. With the peer’s ephemeral public value in hand, both sides can calculate the final or forward-secure keys for the connection. 

Upon sending an SHLO message, the server immediately switches to sending packets encrypted with the forward-secure keys.
Upon receiving the SHLO message, the client switches to sending packets encrypted with the forward-secure keys. 

QUIC’s cryptography therefore provides two levels of secrecy: initial client data is encrypted using initial keys, and subsequent client data and all server data are encrypted using forward-secure keys. The initial keys provide protection analogous to TLS session resumption with session tickets [60]. The forward-secure keys are ephemeral and provide even greater protection. The client caches the server config and source-address token, and on a repeat connection to the same origin, uses them to start the connection with a complete CHLO. As shown in Figure 4, the client can now send initial-key-encrypted data to the server, without having to wait for a response from the server. Eventually, the source address token or the server config may expire, or the server may change certificates, resulting in handshake failure, even if the client sends a complete CHLO. In this case, the server replies with a REJ message, just as if the server had received an inchoate CHLO and the handshake proceeds from there.
```