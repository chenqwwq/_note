# HTTP 协议



HTTP（超文本传输协议），意思就是可以传输文本之外的任何数据，包括图片，音视频等，属于最常见的 C/S 结构协议（C/S 协议就是客户端/服务端协议，有明显的请求和响应



## Http 报文格式

### 请求报文格式

 | 请求方法（GET/POST/） ｜ 请求 URI（/api // /file） ｜ 协议版本（HTTP 1.1）

 | 空行 

｜ 请求头 

 | 空行

｜请求体 



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

另外还有 DELETE，PUT 等（不知道是不是基于 RESTFul 定义的，有些框架还是会默认按照 POST 去处理。



## Header

Header 分为请求头和响应头，分别对应了客户端和服务端的一些配置项。

部分请求头如下：

| 请求头名称    | 含义                                                         | 示例                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Accpet        | 客户端可以接受的内容格式                                     | application/json                                             |
| Host          | 请求的主机信息                                               | www.baidu.com                                                |
| User-Agent    | 使用的代理信息                                               | Postman/xxx                                                  |
| Range         | 分段传输请求头，Range: <unit>=<range-start>-<range-end>，可包含多个范围 | Range:bytes=0-1024                                           |
| If-Range      | 条件范围获取，满足对应条件则返回对应范围，和 Range 配合使用  | If-Range: <day-name>, <day> <month> <year> <hour>:<minute>:<second> GMT If-Range: <etag> |
| Authorization | 用户的认证信息                                               | Authorization: <type> <credentials>                          |

部分响应头如下：

| 响应头名称       | 含义                                                         | 示例                                                   |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| Content-Length   | 响应体内容长度                                               | Content-Length: 1024                                   |
| Content-Range    | 消息范围                                                     | Content-Range: <unit> <range-start>-<range-end>/<size> |
| Content-Type     | 响应体类型                                                   | Content-Type: application/json,text/html               |
| Content-Encoding | 内容的编码格式                                               | Content-Encoding: gzip                                 |
| Accept-Ranges    | 客户端是否支持范围传输，以及范围传输单位                     | Accept-Ranges: none<br />Accept-Ranges:  bytes         |
| Cache-Control    | 设置请求的缓存策略，是否使用缓存，缓存的超时时间。           | Cache-Control: max-age=120                             |
| ETag             | 当前数据的令牌或者摘要，再断点续传时用于判断对应的文件数据是否修改。 |                                                        |
| last-modified    | 包含服务端内容最后的修改时间，相比 ETag 精确度低，所以值作为一个备用机制 ，配合 If-Modified-Since 或者 If-Unmodified-Since 使用 |                                                        |





## 条件请求

条件式的请求就是指服务端会**根据特定的条件请求头判断条件是否成立（绝大部分都是判断资源版本是否匹配）而决定最终的返回结果**。

可以用于缓存是否过期的判断，以及断点续传前对资源是否改变的判断。

<br>

针对不同的请求类型条件请求有不同的行为方式，如果是 GET 请求是在条件满足之后才会满足，如果是 POST 则是条件满足之后才会更新。

<br>

判断条件基本就是资源是否已经改变，判断资源是否改变的根据就是 ETag 或者 last-modified。

相关请求头：

| 请求头              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| If-Match            |                                                              |
| If-None-Match       |                                                              |
| If-Modified-Since   |                                                              |
| If-Unmodified-Since |                                                              |
| If-Range            | 适用于 ETag / last-modified，只有该头部满足后 Range 头部才会生效 |



##  范围请求与断点续传

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

### 分块传输

分块传输和分段下载功能类似，但是分段传输是客户端主动请求的（客户端可以通过 Content-Range 知道具体的内容大小。

但是存在某一些数据无法知道具体的文件大小，此时就是用分快传输。

**涉及到的请求头为： Transfer-Encoding: chunked**

分块传输使用一个空的数据块作为结束标志。

### reference

- [HTTP Range - 分段下载](https://www.dalvik.work/2021/12/16/http-range/)

- [HTTP 条件请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Conditional_requests)



## HTTP 缓存

**HTTP 缓存就是指在客户端缓存响应内容，由响应体中的 Cache-Control 控制缓存的周期以及访问权限。**

<br>

Cache-Control: max-age=120  // 表示缓存的最久保存时间，在这之前不需要重新请求

Cache-Control: no-store    // 不允许保存缓存，每次都会进行数据的传输，例如网页数据中，js、css 等也会重新传输

Cache-Control: no-cache    // 使用缓存，但会根据 ETag 确保当前缓存的有效性

Cache-Control: private｜public // 私有/公有缓存，是否允许中间代理等服务缓存



## Reference

- [W3C HTTP 协议规范 RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616.html)