## 什么是TCP/IP协议
`TCP/IP协议` 不是一个协议而是一个协议族的统称，里面包括了IP协议，IMCP协议，TCP协议等等。同时是Internet最基本的协议、Internet国际互联网络的基础，由网络层的IP协议和传输层的TCP协议组成。 `TCP/IP协议` 定义了电子设备如何连入因特网，以及数据如何在它们之间传输的标准。协议采用了4层的层级结构，每一层都通过调用它的下一层所提供的协议来完成自己的需求，如下图：

![tcp-ip-stack-layer](https://raw.githubusercontent.com/liexusong/tcp-ip-stack/master/images/tcp-ip-stack-layer.jpg)

上图左边对应的是OSI模型分层，OSI把网络按照层次分为七层，由下到上分别为： `物理层`、`数据链路层`、`网络层`、`传输层`、`会话层`、`表示层`、`应用层`。而TCP/IP模型分层只分为四层，分别为：`网络接口层`、`网络层`、`传输层`、`应用层`，它们之间的对应关系如上图所示。



