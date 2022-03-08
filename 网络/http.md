# HTTP 协议



HTTP（超文本传输协议），意思就是可以传输文本之外的任何数据，包括图片，音视频等，属于最常见的 C/S 结构协议（C/S 协议就是客户端/服务端协议，有明显的请求和响应的区别。



## HTTP 报文格式

### 请求报文格式

 | 请求方法（GET/POST/） ｜ 请求 URI（/api // /file） ｜ 协议版本（HTTP 1.1）

 | 空行 

 | 请求头 

 | 空行

 | 请求体 



> 一些 HTTP 的协议实现中，不会解析 GET 请求的请求体。
>
> 请求头是 Name: Value 的形式，可能涉及大量的请求头，导致整个报文过大，所以在 HTTP2.0 中出现了请求头压缩的功能



### 响应报文格式

｜协议版本（HTTP 1.1）｜ 响应码（200） ｜ 响应信息（OK） ｜

｜空行

｜响应头

｜空行

｜ 响应体



> 对于纯 HTTP 的协议报文来说，**中间传输都是明文的**，HTTP 一定程度上规定了传输的格式，使用请求头做 C/S 的功能扩展。



## 请求方法

HTTP 中常见的请求只有 GET/POST，GET 表示获取某些数据，而 POST 则是修改或者新增信息。

额外的还有 HEAD，只用于获取服务端的响应头信息，例如在下载前使用 HEAD 获取响应头，并根据返回判断是否可以分段下载（Accept-Ranges: bytes，表示可以根据 byte 为单位进行分段传输。

另外还有 DELETE，PUT 等（不知道是不是基于 RESTFul 定义的，有些框架还是会默认按照 POST 去处理 DELETE 等请求类型。



## URI 统一资源定位符

URI 用于在 WEB 世界中定位某个资源地址。

```textile
<scheme>://<user>:<password>@<host>:<port>/<path>;<params>?<query>#<fragment>
```

简单的地址例如：https://www.baidu.com



## Header（头部信息）

Header 分为请求头和响应头，分别对应了客户端和服务端的一些配置项。

### 请求首部

| 请求头名称    | 含义                                                         | 示例                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Accpet        | 客户端可以接受的内容格式                                     | application/json                                             |
| Host          | 请求的主机信息                                               | www.baidu.com                                                |
| User-Agent    | 使用的代理信息                                               | Postman/xxx                                                  |
| Range         | 分段传输请求头，Range: <unit>=<range-start>-<range-end>，可包含多个范围 | Range:bytes=0-1024                                           |
| If-Range      | 条件范围获取，满足对应条件则返回对应范围，和 Range 配合使用  | If-Range: <day-name>, <day> <month> <year> <hour>:<minute>:<second> GMT If-Range: <etag> |
| Authorization | 用户的认证信息                                               | Authorization: <type> <credentials>                          |



### 响应首部

| 响应头名称        | 含义                                                         | 示例                                                   |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| Content-Length    | 响应体内容长度                                               | Content-Length: 1024                                   |
| Content-Range     | 消息范围                                                     | Content-Range: <unit> <range-start>-<range-end>/<size> |
| Content-Type      | 响应体类型                                                   | Content-Type: application/json,text/html               |
| Content-Encoding  | 内容的编码格式                                               | Content-Encoding: gzip                                 |
| Accept-Ranges     | 客户端是否支持范围传输，以及范围传输单位                     | Accept-Ranges: none<br />Accept-Ranges:  bytes         |
| Cache-Control     | 设置请求的缓存策略，是否使用缓存，缓存的超时时间。           | Cache-Control: max-age=120                             |
| ETag              | 当前数据的令牌或者摘要，再断点续传时用于判断对应的文件数据是否修改。 |                                                        |
| Last-Modified     | 包含服务端内容最后的修改时间，相比 ETag 精确度低，所以值作为一个备用机制 ，配合 If-Modified-Since 或者 If-Unmodified-Since 使用 |                                                        |
| Transfer-Encoding | 表示传输的格式，基本只用于 chunked 传输                      |                                                        |





## 响应体大小

> 涉及的请求/响应头：
>
> - **Content-Length**
>
> - **Transfer-Encoding**

客户端在接收到响应数据之后如何才能判断响应体是否已经结束。

HTTP 1.1 之前，如果采用非持久化连接，那么连接断开就表示数据传输完毕，可以直接确定报文的数据范围，并不会造成混乱。

HTTP 1.1 之后都会采用持久的 TCP 连接来发送请求并接收响应，对于 TCP 而言所有的数据都是以流的形式存在的，因此连续的响应报文可能就不知道该如何分割，此时就需要 HTTP 客户端自行判断响应体大小（当然也许需要服务端配合。

响应头 Content-Length 就是用来告诉客户端当前的响应体大小的（单独指响应体，突然发现 TOMCAT 里面工作线程只读取到 Header 就先进行处理也是有道理的。

但是存在部分情况可能没办法第一时间获取 Content-Length，比如通过 JSP 渲染的页面，并不是一个确定的值，此时就只能使用 Transfer-Encoding 请求头来表示分块传输。

分块传输使用 **的请求头为： Transfer-Encoding: chunked** ，**并且一个空的数据块作为结束标志。**

[HTTP Range - 分段下载](https://www.dalvik.work/2021/12/16/http-range/)





##  范围请求与断点续传

> 涉及请求/响应头：
>
> - **Range**
>
> - **Content-Range**
>
> - **If-Range**

**范围请求就是针对某个资源，可以请求资源某个部分的数据（所以也可以称为分段请求。**

HTTP 的范围请求主要使用了 **Content-Range（响应头）以及 Range（请求头）**。

Content-Range 中包含范围的表示单位，当前数据片段以及所有数据的大小，例如 Content-Range: bytes-1024，表示资源总长度为1024字节。

Range 则表示请求的数据范围，例如 Range:bytes=0-1024，表示请求 0~1024 字节的数据。

范围请求对应的响应码变为 206 Partial Content，如果范围异常则返回 416 Range Not Satisfiable，如果不支持分段传输，则返回 200 OK。

另外分段传输的形式下，Content-Type 也会变为 multipart/XXX。

<br>

如果在一开始需要判断服务端是否可以进行范围请求，可以使用 HEAD 请求，然后查看响应体中的 **Accept-Range** 头部。

Accept-Range 表示服务端是否支持反范围请求，返回 Accept-Range: bytes 即表示支持以字节为单位的范围请求，如果是 Accept-Range: none 则表示不支持范围请求。

<br>

范围请求在一定程度上支持断点续传的功能，在首次请求经过一段时间之后，可以再次通过范围请求获取余下的数据。

在重新获取的时候需要判断之前的数据是否已经改变（**通过 If-Range 判断 ETag 或者 last-modified。**





## 条件请求

> 涉及请求头：
>
> - **If-Match**
> - **If-None-Match**
> - **If-Modified-Since**
> - **If-Unmodified-Since**
> - **If-Range**
> - **E-Tag**
> - **last-modify**

条件式的请求就是指服务端会**根据特定的条件请求头判断条件是否成立（绝大部分都是判断资源版本是否匹配，通过  ETag 或者 last-modified）而决定最终的返回结果**。

可以用于缓存是否过期的判断，以及断点续传前对资源是否改变的判断。

<br>

针对不同的请求类型条件请求有不同的行为方式，如果是 GET 请求是在条件满足之后才会满足，如果是 POST 则是条件满足之后才会更新。

<br>

判断条件基本就是资源是否已经改变，**判断资源是否改变的根据就是 ETag 或者 last-modified。**

相关请求头：

| 请求头              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| If-Match            |                                                              |
| If-None-Match       |                                                              |
| If-Modified-Since   |                                                              |
| If-Unmodified-Since |                                                              |
| If-Range            | 适用于 ETag / last-modified，只有该头部满足后 Range 头部才会生效 |

[HTTP 条件请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Conditional_requests)





## HTTP 缓存

> 涉及请求头：
>
> - **Cache-Control**

**HTTP 缓存就是指在客户端缓存响应内容，由响应体中的 Cache-Control 控制缓存的周期以及访问权限。**

<br>

Cache-Control: max-age=120  // 表示缓存的最久保存时间，在这之前不需要重新请求

Cache-Control: no-store    // 不允许保存缓存，每次都会进行数据的传输，例如网页数据中，js、css 等也会重新传输

Cache-Control: no-cache    // 使用缓存，但会根据 ETag 确保当前缓存的有效性

Cache-Control: private｜public // 私有/公有缓存，是否允许中间代理等服务缓存





### HTTP Pipeline（管线化）

管线化是在 HTTP 1.1 版本引入的机制。

初始的 HTTP 协议属于完全的停止等待模式，后一个请求只有在前一个请求的响应回来之后才可以发送，这就导致了严重的请求延时问题（包括队头阻塞问题。

> 队头阻塞是指在前面的响应时间变大甚至阻塞之后，后续请求也被阻塞。

管线化机制可以跳过等待响应的时候，直接发送后续的请求报文，从而减少总体的网络延时。

但是问题在于管线化并没有解决响应乱序问题，请求相当于使用 FIFO 的队列保存，客户端直接发送多个请求而希望服务端可以按照顺序回传响应。

**管线化无法解决队头阻塞问题。**





## HTTP 首部压缩（Header Compress）

首部压缩是在 HTTP 2.0 中使用的新特性，因为在 HTTP 的 Request / Response 中往返存在很多相同的请求头。

（比如，connection: keep-alive 这种每次请求都会传输，这就是带宽的损耗。

HTTP 2.0 中首部只需要在首次请求时添加到请求中，之后仅仅在修改的时候才需要改变。

<br>

HTTP 的首部压缩是和二进制分帧同时推出的。





## HTTP 二进制分帧

（同样是 HTTP 2.0 中提出的新机制。

HTTP 2.0 中数据内容改为了二进制传输（HTTP 2.0 之前都是以文本格式传输的（甚至明文），并新增了一层二进制分帧层，来实现从 1.x 版本的明文传输到二机制传输的过渡。

<br>

![http2_binary_framing_layer](https://st.imququ.com/i/webp/static/uploads/2015/05/http2_1.png.webp)



在二进制分帧层种，一个 TCP 连接被分为多个流（Stream），一个报文也会被分为多个帧（Frame）被分开传输，但是每个帧都会打上对应的流标记，响应的报文也会被同样打上标记，在该层被重新组装成一个完整的报文，继续往上传递。

<br>

HTTP 2.0 所有的请求都可以被称为消息，消息经过二进制分帧层之后被转化为 n 个帧（一个请求可能被分解为多个帧），帧结构又可以分为 Headers Frame 以及 Data Frame 两种，分别编码和传输。

同一个请求分出来的帧以及其对应的响应帧在一个流中被处理（分帧的时候会打上流的标记，对应的处理回传的响应帧。

流是对连接的进一步分解，一个 TCP 连接中可以分出多个流分别处理各自的请求和响应（相当于对连接的多路复用。

另外就是在请求中可以包含优先级参数，并且因为流是可以双向传输数据源的，所以在二进制分帧层中可以包含流量控制以及服务端推送的功能。





[HTTP/2 与 WEB 性能优化（二）](https://imququ.com/post/http2-and-wpo-2.html)



## HTTP 各版本更新

### HTTP 0.9

- 请求类型只有 GET。

- 请求结束之后 TCP 连接断开（不复用。

- 使用 CRLF 作为协议结尾。



### HTTP 1.0 

- 响应增加状态行

例如，HTTP1.0 200 OK

- 增加 HTTP 首部，并扩展传输内容（通过首部协商
- 扩展请求方法（POST
- 请求结束后 TCP 连接依旧会断开



### HTTP 1.1

- 持久化连接（连接复用

通过 connection: keep-alive 来通知服务端希望保持连接状态

- 增加 chunked 传输方式，用于传输大小未知的响应体

增加 Transfer-Encoding: chunked 

- 增加 HTTP 的缓存机制（RFC 7234），范围请求（RFC 7233），条件请求（RFC 7232）





### HTTP 2.0（RFC 7540）

- 二进制分帧（多路复用

HTTP 2.0 之后报文不在于文本形式明文传输，而改为二进制传输。

-  首部压缩（Headers Compress RFC7541）

通过双端的首部表使首部在不改变的情况下只传输一次。

- 服务端推送（Server Push）

服务端可以直接推送数据到客户端，比如请求页面的时候可以直接推送关联的 JS 文件。



## 其他

在 rfc2616 中规定，一个浏览器针对一个域名最多只能发起两个 TCP 连接。



## Reference

- [W3C HTTP 协议规范 RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616.html)