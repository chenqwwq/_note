# Netty 中的编解码逻辑



---

[TOC]

---

## 概述

编解码是网络编程中必不可少的一个步骤，因为网络中传输的都是01的bit位，体现在 Java Nio 中也是各类的 Buffer，底层就是 byte 数组。

我们不可能直接对着 Buffer 实现自己的业务逻辑，所以就需要编解码来实现 字节数组 到 Java对象 的转换。

笼统来说，Netty 中提供了以下两大类编解码器：

1. ByteToMessageDecoder / MessageToByteEncoder  -  字节数组到协议对象的编解码器 
2. MessageToMessageDecoder / MessageToMessageEncoder  -  协议对象到另外种协议对象的编解码器

> Encoder 和 Decoder 的区别：
>
> 1. Encoder 负责的是出站数据，而 Decoder 负责的是入站数据。
> 2. Encoder 实现的是 write，而 Decoder 实现的 channelRead 或者 channelComplete 
> 3. Encoder 的原始数据一般是 Java 对象，也就是 Message，而 Decoder 的原始数据一般是 ByteBuf，不过也有 MessageToMessageDecoder。



## Netty 中对编解码的约束

> **粗略来说，Netty 中要求编码的最终结果就是 ByteBuf，并且也是以 ByteBuf 作为解码的起点。** 

整个 Netty 的写数据流程，在 AbstractChannel 的 write 方法实现中有这么一步:

```java
int size;
try {
    // 对出站消息的拦截过滤
    msg = filterOutboundMessage(msg);
    size = pipeline.estimatorHandle().size(msg);
    if (size < 0) {
        size = 0;
    }
} catch (Throwable t) {
    safeSetFailure(promise, t);
    ReferenceCountUtil.release(msg);
    return;
}
```

所有出站的消息都会经过 filterOutboundMessage 的过滤。

例如使用 NioSocketChannel#writeAndFlush 方法写入的数据，最终都会在其父类也就是 AbstractNioByteChannel 中被过滤一遍。

```java
# AbstractNioByteChannel#filterOutboundMessage
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }

        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
        "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```

**从以上方法很容易看出来，写入的数据只接收 ByteBuf 或者 FileRegion，并且如果是 JVM 内存还会被转化为 Direct 也就是直接内存。**

> 除去 FileRegion 不说，也就是正常的数据都要经过 ByteBuf 来输出。

> Q: 为什么 ByteBuf 需要转化为直接  DirectBuffer ?
>
> A: 不确定，但是一定程度上还是因为要写数据，JVM 的逻辑地址在GC等情况下容易发生改变，所以需要较为稳定的直接内存。
>
> **在正常的 IO 处理中其实也会经过这么一步。**



有一个 read 操作是意外就是，NioServerSocketChannel 的 read 方法

以上是写数据的流程，而读数据的开端就是在 NioEventLoop#run 里面了，几层调用后会到达 AbstractNioByteChannel#read 方法。

以下是循环读取的主要逻辑:

```java
do {
    byteBuf = allocHandle.allocate(allocator);
    allocHandle.lastBytesRead(doReadBytes(byteBuf));
    if (allocHandle.lastBytesRead() <= 0) {
        // nothing was read. release the buffer.
        byteBuf.release();
        byteBuf = null;
        close = allocHandle.lastBytesRead() < 0;
        if (close) {
            // There is nothing left to read as we received an EOF.
            readPending = false;
        }
        break;
    }
    allocHandle.incMessagesRead(1);
    readPending = false;
    pipeline.fireChannelRead(byteBuf);
    byteBuf = null;
} while (allocHandle.continueReading());
```

**逻辑并不复杂，每次都分配一定的 ByteBuf 空间，然后读取到数据后通过 fireChannelRead 方法传递出去。**

所以，此时处理入站数据的起点就是一个 ByteBuf。



如果简单的讨论 read 方法，这里还有一个意外，就是 NioServerSocketChannel#read。

> **NioServerSocketChannel#read 方法实际上响应的是 ACCPET 事件，读取的就是 NioSocketChannel 对象。**



> ！！！编解码器是一种相对特殊的 ChannelHandler。
>
> 因为 ChannelPipeline 的线性执行，所有编解码器的声明顺序也很关键。
>
> 声明在后的解码器处理的就是前面的解码结果，而声明在后的编码器，它的结果会作为前面的编码器入参。





## MessageToByteEncoder

> Encoder 就是将 Java 对象编码成 ByteBuf 的过程，对应的就是实现类似 write 或者 writeAndFlush 方法。

以下就是 MessageToByteEncoder#write 的全部源码：

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        // 检查是否属于自己编码的类型
        if (acceptOutboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            // 分配空间
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                // 编码
                // encode 是编码方法，会传入对象以及buf
                // buf 就是编码后的数据存储的地方
                encode(ctx, cast, buf);
            } finally {
                ReferenceCountUtil.release(cast);
            }

            if (buf.isReadable()) {
                // 直接写
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        // 写完之后释放空间
        if (buf != null) {
            buf.release();
        }
    }
}
```



> 编码的过程好像很简单：
>
> 1. 判断接受的对象是否需要编码
> 2. 分配一定的空间
> 3. 调用模板方法进行编码

模板方法就是自定义编码器需要实现的部分，就是自己序列化对象为 byte 数组，然后塞入入参的 ByteBuf 就可。

**acceptOutboundMessage 方法判断的就是是否为当前类的泛型类型，只要是那么就会进入编码过程，所以如果涉及的写入对象有多种类型，可能就需要各自实现编码方式。**



> MessageToMessageEncoder 和 MessageToByteEncoder 的逻辑差不多。
>
> 后者使用的 ByteBuf 接收编码后的数据，而前者使用 CodecOutputList 来接收编码后的数据。



## ByteToMessageDecoder 

> Encoder 好理解一点，但是解码部分还存在分包和粘包等等问题，所以会稍微麻烦一点。
>
> 主要还是看如何解决分包和粘包的问题。

以下是 ByteToMessageDecoder#channelRead 部分：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	// ByteToMessageDecoder 只解析 ByteBuf，对于非 ByteBuf 对象直接就往后传递了
    if (msg instanceof ByteBuf) {
        // 分配一个列表,CodecOutputList 是 Netty 自身实现的一个带生命周期的List
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            // cumulatio 是类变量 first就表示是否是首次使用Decoder
            first = cumulation == null;
            // 累加器
            cumulation = cumulator.cumulate(ctx.alloc(),
                                            first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
            // 解码，解码之后的数据会放到out中
            // 这里 cumulation 就是希望解码的 ByteBuf，是多次的读取累加起来一起的
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            try {
                // 已经读完了就释放所有的 ByteBuf
                // 就将占用的 ByteBuf 释放
                if (cumulation != null && !cumulation.isReadable()) {
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                // 下面的是为了预防 OOM 的，在读取之后一直累加的的化也会要出现问题
                // 再读取了 discardAfterReads 也就是 16 读取后将readIndex归零
                } else if (++numReads >= discardAfterReads) {
                    numReads = 0;
                    discardSomeReadBytes();
                }

                int size = out.size();
                firedChannelRead |= out.insertSinceRecycled();
                // 将解码后的对象往后传
                fireChannelRead(ctx, out, size);
            } finally {
                out.recycle();
            }
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}

```

**以上的源码好理解，主要逻辑还是在累加器的作用以及 callDecode 方法的调用。**

> Q: 累加器的作用？
>
> A: 主要是应对 TCP 的流式特性，上层应用的一段数据可能会被拆分成多个不同的报文或者多段数据组合成一个包，也就出现了所谓分包，半包或者粘包的现象，如果以一个数据报为单位解码就可能出现数据错乱的问题，**前后累加就是以接受窗口的形式处理整体数据。**
>
> 不过前后累加也需要注意及时的释放内存，后续可以关注一下，Netty 怎么判断数据已经被读取，怎么释放累加器的内存。

> ！！！注意就是所有的 ByteToMessageDecoder 的实现都不能是 Shareable 的，因为存在成员变量 first.





累加器会将前后的数据报组合到一起，之后再传给解码器：

```java
// out 就是解码后的对象列表
// in 就是解码的数据，传入的累加器的 ByteBuf
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        // 调用都是循环的,在 ByteBuf 不可读之前
        while (in.isReadable()) {
            int outSize = out.size();
			// 有对象解出来
            if (outSize > 0) {
                // 继续往后传递
                fireChannelRead(ctx, out, outSize);
                // 清空ByteBuf
                // 但是此时没有跳出外层的 while 循环
                out.clear();
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
            // 经过以上的步骤,outSize 肯定等于0了
            // 记录可读的长度
			int oldInputLength = in.readableBytes();
            // 解码,解码之后的对象都会保存在 out 的链表中
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }
            // 表示有东西解出来
            // 如果 outSize 等于0,表示
            if (outSize == out.size()) {
                // 
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                    StringUtil.simpleClassName(getClass()) +
                    ".decode() did not read anything but decoded a message.");
            }
            if (isSingleDecode()) {
                break;
            }
        }
...
```

因为累加器中可能包含多个对象所以使用的是 List<Object> 接收解码后的对象。

在 ByteBuf  没有可读数据或者没有解析出数据的时候退出循环。



> 以上那个 outSize = 0 的赋值感觉上很多余，哈哈哈 顺手提了pr。



> Q: 整个 ByteToMessage 的编码流程
>
> 1. 读取数据并添加到累加器
> 2. 将累加器数据传入解码器解码
> 3. 解码之后的数据通过 channelRead 事件继续向后传递

>注意，解码的对象不是单个的数据报，而是累加器中的数据，原因上面说过
>
>另外，MessageToMessageDecoder 的逻辑也类似。



## Netty 中提供了特定的编解码器

> Netty 提供了几种固定的编解码器：
>
> 1. LengthFieldBasedFrameDecoder - 以报文长度来分割报文
> 2. LineBasedFrameDecoder - 以换行符来分割报文
> 3. StringDecoder / StringEncoder
> 4. 还有 Protobuf 或者 xml 格式的编解码器。



## Netty 如何处理分包和粘包

> 分包和粘包的原因就是因为 TCP 的流式传输特性。

解决分包和粘包的方法有很多，常用的有以下几种：

### 定长报文

固定报文的长度，以固定的长度解析报文，如果写入报文不够就填充固定字符。

**在 Netty 中以 FixedLengthFrameDecoder 实现，**缺点也明显填充的形式来满足固定的报文长度有点浪费内存空间。



### 固定分隔符

已分隔符作为一个报文的结束，每次读取读到分隔符，作为结束，常规的分隔符有 '\n'，'\r' 等。

Netty 中实现了 LineBasedFrameDecoder 以换行符作为分割符，DelimiterBasedFrameDecoder 以指定的特殊字符作为分隔符。

缺点也明显，分隔符需要保证绝对的唯一性。

一般来说，上层的应用不应该关心传输的时候，而使用该方法之后会一定程度上反向限制上层应用，所以也不是很推荐。



### 报文长度属性

在报文中以整个报文长度作为开端，读取满指定长度之后就结束，剩下的就是下个报文的内容。

实现的方法很简单，在发送前会获取报文的相关长度并填充到头部，解析时以该长度为参考读取一定大小的数据。

Netty 中实现了 LengthFieldBasedFrameDecoder 作为解码器，LengthFieldPrepender 作为编码器。



####  LengthFieldPrepender 类简析

以下是 LengthFieldPrepender#encoder 的源码：

![image-20210506232514546](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210506232514546.png)

根据指定的长度属性所占的字节数（lengthFieldLength）分配内存并写入长度，之后通过 `add(msg.retain())` 组合原报文。

**简单来说，LengthFieldPrepender 是在全部的报文之前增加一个固定长度的报文长度属性。**



#### LengthFieldBasedFrameDecoder 类简析

> TODO:

比如自定义协议格式的时候，可能请求头长度固定，负载的长度在中间，此时就无法使用 LengthFieldPrepender，但是解码依旧可以使用 LengthFieldBasedFrameDecoder。





