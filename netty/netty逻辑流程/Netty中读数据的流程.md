# Netty 中数据的读取流程

---

[TOC]

---





<br>

## 概述

Netty 的数据读取可以分为以下两类：

- OP_ACCPET  -  该类型在在服务端 Channel 触发，表示有新连接
- OP_READ  -  该类型在客户端 Channel 触发，表示客户端有新数据

下文主要关注的就是 OP_READ 事件的处理。





## 源码实现

```java
// AbstractNioByteChannel$NioByteUnsafe#read
@Override
public final void read() {
    System.out.println("NioByteUnsafe#read()");
    final ChannelConfig config = config();
    // 处理半连接相关内容，判断输入是否已经断开，是否允许半连接
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        /**
                 * 以下可以当做是客户端或者workerGroup的读取逻辑，
                 * 和bossGroup相比，这里是读取一个处理一个的,在fireChannelRead的调用链中
                 * 如果没有指定另外的线程池，则直接使用IO线程处理相应逻辑，如果有的话则调用指定的线程池执行
                 * 这个还是很关键的，在Pipeline的执行过程中，如果有阻塞任务或者比较耗时的任务,会直接影响整个EventLoop的运行
                 * 默认的workerGroup是CPU核数个EventLoop，卡死一个影响还蛮大的。
                 */
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

        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
}
```





## 总结

read 的过程就是读取数据并且使用 fireChannelRead 传递出去。

主要的优化在内存的时候，会使用内存池相关实现，默认是 PooledByteBufAllocator。