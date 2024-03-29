# tcp



#### 为什么三次握手？

第一次握手：A向B发送信息之后，B收到信息后可以确认自己的收信能力和A的发信能力没有问题。

第二次握手：B向A发送信息之后，A可以确认自己的发信能力和B的收信能力没有问题，但是B不知道自己的发信能力到底如何，所以就需要第三次通信。

第三次握手：A向B发送信息之后，B就可以确认自己的发信能力没有问题。

3次握手完成两个重要的功能，`既要双方做好发送数据的准备工作(双方都知道彼此已准备好)`，也要允许双方就初始序列号进行协商，这个序列号在握手过程中被发送和确认。

主要原因：

为了防止服务器端开启一些无用的连接增加服务器开销以及防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误。

客户端可能设置了一个超时时间，时间到了就关闭了连接创建的请求。再重新发出创建连接的请求，而服务器端是不知道的，如果没有第三次握手告诉服务器端客户端收的到服务器端传输的数据的话，

服务器端是不知道客户端有没有接收到服务器端返回的信息的。

![img](https://pics3.baidu.com/feed/1c950a7b02087bf4cf316c92aa0daf2913dfcfd4.jpeg?token=7e26ac525676d063251da3d6bf395e8a&s=3172483221D25DCA14F115DA0300E0B0)

这样没有给服务器端一个创建还是关闭连接端口的请求，服务器端的端口就一直开着，等到客户端因超时重新发出请求时，服务器就会重新开启一个端口连接。那么服务器端上没有接收到请求数据的上一个端口就一直开着，长此以往，这样的端口多了，就会造成服务器端开销的严重浪费。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020031913392621.png)



#### 为什么四次挥手？

![img](https://pics5.baidu.com/feed/48540923dd54564e5260495ce0006487d0584fb6.jpeg?token=c3a743af38e25ff66deb6a07891be58e&s=C584FC1A71CFF4EE1A75A45203007073)

![img](https://pics3.baidu.com/feed/caef76094b36acaf042ba27e2f07751503e99c48.jpeg?token=82e7f4e96e77dc4f3a6b5d9d1e36af2c&s=C150C53249BAC4CA586931D6030050B2)







#### 粘包/拆包 问题产生原因：

发生TCP粘包或拆包有很多原因，现列出常见的几点：

- 要发送的数据大于TCP发送缓冲区剩余空间大小，将会发生拆包。
- 待发送数据大于MSS（TCP报文长度 - TCP头部长度 > MSS最大报文长度），TCP在传输前将根据MSS大小进行拆包分段发送。
- 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据包合并为一次发送，将发生粘包（Nagle算法优化，避免tcp报文头重脚轻的情况发生）
- 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包。
  

定长报文或者分隔符处理