# ByteBuf

## ByteBuffer

缺点：内存固定且 API 略复杂（还要 flip 啥的，妈哟）。



Netty 针对 ByteBuffer 改进了其复杂的 API，使用 readIndex 和 writeIndex 分别表示读写偏移量。