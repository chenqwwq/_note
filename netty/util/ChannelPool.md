# ChannelPool



> ChannelPool 是 Netty 对 Channel 的池化实现。



## 初步阅读

1. ChannelPool 只有在获取的时候才有健康检查
2. ChannelPool 的创建需要传入 Bootstrap，但是 Bootstrap 的 ChannelHandler 需要通过 ChannelPoolHandler 的 channelCreated 方法设置。
3. 可以通过参数设置 Promise 或者通过返回值设置 Future

> Promise 和 Future 相比