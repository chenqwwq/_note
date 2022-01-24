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



###







## Http 断点续传

短点续传功能的定义依靠的是 HTTP1.1 中定义的 Range 和 Content-Range。

Range 作为请求体指明需要的数据范围，而 Content-Range 作为响应体返回传输的数据范围，另外此时的响应码（Response Code）应该是 206（Partial Content。

另外续传还有对原始文件的校验，如果文件发生改动则需重新开始下载。

RFC 2616 中定义了 If-Range 以之前的 ETag（Response Header）作为参数，如果匹配则进行续传。







## Header

Header 分为请求头和响应头。

### Request Header



#### Accpet

客户端可以接受的



### Response Header



#### Content-Length 

响应体（ResponseBody）长度。

```
// eg:
Content-Length: 1024
```

#### Cache-Control

设置请求的缓存策略，是否使用缓存，缓存的超时时间。

```
// eg:
Cache-Control: max-age=120 // 表示缓存的最久保存时间，在这之前不需要重新请求
Cache-Control: no-store    // 不允许保存缓存，每次都会进行数据的传输，例如网页数据中，js、css 等也会重新传输
Cache-Control: no-cache    // 使用缓存，但会根据 ETag 确保当前缓存的有效性


Cache-Control: private｜public // 私有/公有缓存，是否允许中间代理等服务缓存

```

#### ETag

当前数据的令牌或者摘要，再断点续传时用于判断对应的文件数据是否修改。



#### Content-Encoding

内容的编码格式。

```
// eg:
Content-Encoding: gzip
```







## Reference

- [W3C HTTP 协议规范 RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616.html)