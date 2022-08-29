### 1、堵塞io的读写方式

一次读写过程

```
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

首先，期间共发生了 4 次用户态与内核态的上下文切换，因为发生了两次系统调用，一次是 read() ，一次是 write()，每次系统调用都得先从用户态切换到内核态，等内核完成任务后，再从内核态切换回用户态。

上下文切换到成本并不小，一次切换需要耗时几十纳秒到几微秒，虽然时间看上去很短，但是在高并发的场景下，这类时间容易被累积和放大，从而影响系统的性能。

其次，还发生了 4 次数据拷贝，其中两次是 DMA 的拷贝，另外两次则是通过 CPU 拷贝的，下面说一下这个过程：

第一次拷贝，把磁盘上的数据拷贝到操作系统内核的缓冲区里，这个拷贝的过程是通过 DMA 搬运的。
第二次拷贝，把内核缓冲区的数据拷贝到用户的缓冲区里，于是我们应用程序就可以使用这部分数据了，这个拷贝到过程是由 CPU 完成的。
第三次拷贝，把刚才拷贝到用户的缓冲区里的数据，再拷贝到内核的 socket 的缓冲区里，这个过程依然还是由 CPU 搬运的。
第四次拷贝，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程又是由 DMA 搬运的。
这种简单又传统的文件传输方式，存在冗余的上文切换和数据拷贝，在高并发系统里是非常糟糕的，多了很多不必要的开销，会严重影响系统性能。

所以，要想提高文件传输的性能，就需要减少「用户态与内核态的上下文切换」和「内存拷贝」的次数。

![img](https://img-blog.csdnimg.cn/img_convert/f31ece783dd86621734188fa2d58b4bb.png)

### 2、零拷贝技术原理

零拷贝主要是用来解决操作系统在处理 I/O 操作时，频繁复制数据的问题。关于零拷贝主要技术有 `mmap+write`、`sendfile`和`splice`等几种方式。

#### （1）mmap+write

虚拟内存

- 多个虚拟内存可以指向同一个物理地址。
- 虚拟内存空间可以远远大于物理内存空间。

利用上述的第一条特性可以优化，可以把内核空间和用户空间的虚拟地址映射到同一个物理地址，这样在 I/O 操作时就不需要来回复制了。

![image-20210812181924274](https://img-blog.csdnimg.cn/img_convert/c4b54a3fdbb75412d34d8648a1063c65.png)

使用`mmap/write`方式替换原来的传统I/O方式，就是利用了虚拟内存的特性。下图展示了`mmap/write`原理：

![image-20210812201839908](https://img-blog.csdnimg.cn/img_convert/42ecc5a57a0633de251f7fef6f9cc194.png)



上述流程就是少了一个 CPU COPY，提升了 I/O 的速度。不过发现上下文的切换还是4次并没有减少，这是因为还是要应用程序发起`write`操作。

> 那能不能减少上下文切换呢?这就需要`sendfile`方式来进一步优化了。

#### （2）sendfile

从 Linux 2.1 版本开始，Linux 引入了 `sendfile`来简化操作。`sendfile`方式可以替换上面的`mmap/write`方式来进一步优化。

`sendfile`将以下操作：

```java
  mmap();
  write();
```

替换为：

```
sendfile();
```

这样就减少了上下文切换，因为少了一个应用程序发起`write`操作，直接发起`sendfile`操作

![image-20210812201905046](https://img-blog.csdnimg.cn/img_convert/2a99579176ea9bec8a86c2a8bf1f9af8.png)

`sendfile`方式只有三次数据复制（其中只有一次 CPU COPY）以及2次上下文切换。

> 那能不能把 CPU COPY 减少到没有呢？这样需要带有 `scatter/gather`的`sendfile`方式了。





#### （3）带有 scatter/gather 的 sendfile方式

Linux 2.4 内核进行了优化，提供了带有 scatter/gather 的 sendfile 操作，这个操作可以把最后一次 CPU COPY 去除。其原理就是在内核空间 Read BUffer 和 Socket Buffer 不做数据复制，而是将 Read Buffer 的内存地址、偏移量记录到相应的 Socket Buffer 中，这样就不需要复制。其本质和虚拟内存的解决方法思路一致，就是内存地址的记录。

![image-20210812201922193](https://img-blog.csdnimg.cn/img_convert/7f59be15f99196d594a7b065e8dfc7e7.png)

scatter/gather 的 sendfile 只有两次数据复制（都是 DMA COPY）及 2 次上下文切换。CUP COPY 已经完全没有。不过这一种收集复制功能是需要硬件及驱动程序支持的。

#### （4）splice

splice 调用和sendfile 非常相似，用户应用程序必须拥有两个已经打开的文件描述符，一个表示输入设备，一个表示输出设备。与sendfile不同的是，splice允许任意两个文件互相连接，而并不只是文件与socket进行数据传输。对于从一个文件描述符发送数据到socket这种特例来说，一直都是使用sendfile系统调用，而splice一直以来就只是一种机制，它并不仅限于sendfile的功能。也就是说 sendfile 是 splice 的一个子集。

在 Linux 2.6.17 版本引入了 splice，而在 Linux 2.6.23 版本中， sendfile 机制的实现已经没有了，但是其 API 及相应的功能还在，只不过 API 及相应的功能是利用了 splice 机制来实现的。

和 sendfile 不同的是，splice 不需要硬件支持。

#### （5）java对零拷贝的支持

- Java NIO对mmap的支持

Java NIO有一个`MappedByteBuffer`的类，可以用来实现内存映射。它的底层是调用了Linux内核的mmap的API。

mmap的小demo如下：

```
public class MmapTest {

    public static void main(String[] args) {
        try {
            FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
            MappedByteBuffer data = readChannel.map(FileChannel.MapMode.READ_ONLY, 0, 1024 * 1024 * 40);
            FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
            //数据传输
            writeChannel.write(data);
            readChannel.close();
            writeChannel.close();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }
    }
}
```

- Java NIO对sendfile的支持

FileChannel的`transferTo()/transferFrom()`，底层就是sendfile() 系统调用函数。Kafka 这个开源项目就用到它，平时面试的时候，回答面试官为什么这么快，就可以提到零拷贝`sendfile`这个点。

```
@Override
public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {
   return fileChannel.transferTo(position, count, socketChannel);
}
```

sendfile的小demo如下：

```
public class SendFileTest {
    public static void main(String[] args) {
        try {
            FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
            long len = readChannel.size();
            long position = readChannel.position();
            
            FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
            //数据传输
            readChannel.transferTo(position, len, writeChannel);
            readChannel.close();
            writeChannel.close();
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }
}
```

#### （6）总结

无论是传统的 I/O 方式，还是引入了零拷贝之后，2 次 `DMA copy`是都少不了的。因为两次 DMA 都是依赖硬件完成的。所以，所谓的零拷贝，都是为了减少 CPU copy 及减少了上下文的切换。

下图展示了各种零拷贝技术的对比图：

|                     | CPU拷贝 | DMA拷贝 | 系统调用   | 上下文切换 |
| ------------------- | ------- | ------- | ---------- | ---------- |
| 传统方法            | 2       | 2       | read/write | 4          |
| 内存映射            | 1       | 2       | mmap/write | 4          |
| sendfile            | 1       | 2       | sendfile   | 2          |
| scatter/gather copy | 0       | 2       | sendfile   | 2          |
| splice              | 0       | 2       | splice     | 0          |



### 3、nio

NIO有三大核心部分: **Channel(通道)，Buffer(缓冲区)，Selector(选择器)**

**Buffer(缓冲区)**

​    缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。相比较直接对数组的操作，Buffer APl更加容易操作和管理。

**Channel(通道)**

​    Java NIO的通道类似流，但又有些不同:既可以从通道中读取数据，又可以写数据到通道。但流的(input或output)读写通常是单向的。通道可以非阻塞读取和写入通道，通道可以支持读取或写入缓冲区，也支持异步地读写。

**Selector(选择器)**

​    Selector是一个ava NIO组件，可以能够检查一个或多个NIO通道，并确定哪些通道已经准备好进行读取或写入。这样，一个单独的线程可以管理多个channel，从而管理多个网络连接，提高效率

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210323145130359.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3VuaXF1ZV9wZXJmZWN0,size_16,color_FFFFFF,t_70)

- 每个channel都会对应一个 Buffer
- 一个线程对应Selector ,一个Selector对应多个channel(连接)程序
- 切换到哪个channel是由事件决定的
- Selector 会根据不同的事件，在各个通道上切换
- Buffer 就是一个内存块，底层是一个数组
- 数据的读取写入是通过 Buffer完成的，BlO中要么是输入流，或者是输出流,不能双向，但是NIO的Buffer是可以读也可以写。
- Java NIO系统的核心在于:通道(Channel)和缓冲区(Buffer)。通道表示打开到lO设备(例如:文件、套接字)的连接。若需要使用NIO系统，需要获取用于连接IO设备的通道以及用于容纳数据的缓冲区。然后操作缓冲区，对数据进行处理。简而言之，Channel负责传输，Buffer负责存取数据

#### （1）NIO核心一:缓存区 (Buffer)

缓冲区的基本属性 Buffer 中的重要概念：

**容量 (capacity) ：**作为一个内存块，Buffer具有一定的固定大小， 也称为"容量"，缓冲区容量不能为负，并且创建后不能更改。

**限制 (limit)：**表示缓冲区中可以操作数据的大小 （limit 后数据不能进行读写）。缓冲区的限制不能 为负，并且不能大于其容量。 写入模式，限制等于 buffer的容量。读取模式下，limit等于写入的数据量。

**位置 (position)：**下一个要读取或写入的数据的索引。 缓冲区的位置不能为 负，并且不能大于其限制

**标记 (mark)与重置 (reset)：**标记是一个索引， 通过 Buffer 中的 mark() 方法 指定 Buffer 中一个 特定的 position，之后可以通过调用 reset() 方法恢 复到这 个 position.

标记、位置、限制、容量遵守以下不变式： **0 <= mark <= position <= limit <= capacity**

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032315392734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3VuaXF1ZV9wZXJmZWN0,size_16,color_FFFFFF,t_70)

**Buffer常见方法：**

1. **Buffer clear() ：**清空缓冲区并返回对缓冲区的引用

2. **Buffer flip() ：**为 将缓冲区的界限设置为当前位置， 并将当前位置重置为 0

3. **int capacity() ：**返回 Buffer 的 capacity 大小

4. **boolean hasRemaining()：** 判断缓冲区中是否还有元素

5. **int limit() ：**返回 Buffer 的界限(limit) 的位置

6. **Buffer limit(int n)** 将设置缓冲区界限为 n, 并返回一个具有新 limit 的缓冲区对象

7. **Buffer mark()：** 对缓冲区设置标记

8. **int position() ：**返回缓冲区的当前位置 position

9. **Buffer position(int n) ：**将设置缓冲区的当前位置为 n， 并返回修改后的 Buffer 对象

10. **int remaining() ：**返回 position 和 limit 之间的元素个数

11. **Buffer reset() ：**将位置 position 转到以前设置的mark 所在的位置

12. **Buffer rewind() ：**将位置设为为 0， 取消设置的 mark

    

#### （2）NIO核心二：通道(Channel)

##### 1、通道Channe概述

​    通道（Channel）：由 java.nio.channels 包定义 的。Channel 表示 IO 源与目标打开的连接。 Channel 类似于传统的“流”。只不过 Channel **本身不能直接访问数据**，Channel 只能与 Buffer 进行交互。

##### 2、NIO 的通道类似于流，但有些**区别**如下：

- 通道可以同时进行读写，而流只能读或者只能写
- 通道可以实现异步读写数据
- 通道可以从缓冲读数据，也可以写数据到缓冲:

##### 3、BIO 中的 stream 是单向的，例如 FileInputStream 对象只能进行读取数据的操作，而 NIO 中的通道(Channel)是双向的，可以读操作，也可以写操作。

##### 4、Channel 在 NIO 中是一个接口

```java
public interface Channel extends Closeable{}
```

##### 5、常用的Channel实现类

- FileChannel：用于读取、写入、映射和操作文件的通道。
- DatagramChannel：通过 UDP 读写网络中的数据通道。
- SocketChannel：通过 TCP 读写网络中的数据。
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。 【ServerSocketChanne 类似 ServerSocket , SocketChannel 类似 Socket】

#### （3）NIO核心三：选择器(Selector)

##### 1、选择器(Selector)概述

​    选择器（Selector)是**SelectableChannle对象**的**多路复用器**，Selector可以**同时监控多个**SelectableChannel的IO状况，也就是说，利用Selector可使**一个单独的线程管理多个Channel**。Selector是非阻塞IO的核心。

> - Java 的 NIO，用非阻塞的 IO 方式。可以用一个线程，处理多个的客户端连接，就会使用到 Selector(选择器)
> - Selector 能够检测多个注册的通道上是否有事件发生(注意:多个 Channel 以事件的方式可以注册到同一个(Selector)，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用一个单线程去管
> - 理多个通道，也就是管理多个连接和请求。
> - 只有在连接/通道真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程
> - 避免了多线程之间的上下文切换导致的开销

##### 2、选择器的应用

创建 Selector ：通过调用 Selector.open() 方法创建一个 Selector。

```java
Selector selector = Selector.open();
```

向选择器注册通道：SelectableChannel.register(Selector sel, int ops)

```java
//1. 获取通道
ServerSocketChannel ssChannel = ServerSocketChannel.open();


//2. 切换非阻塞模式
ssChannel.configureBlocking(false);

//3. 绑定连接
ssChannel.bind(new InetSocketAddress(9898));

//4. 获取选择器
Selector selector = Selector.open();
//5. 将通道注册到选择器上, 并且指定“监听接收事件”
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
```

当调用 register(Selector sel, int ops) 将通道注册选择器时，选择器对通道的监听事件，需要通过第二个参数 ops 指定。可以监听的事件类型（用 可使用 SelectionKey 的四个常量 表示）：

- **读 : SelectionKey.OP_READ （1）**
- **写 : SelectionKey.OP_WRITE （4）**
- **连接 : SelectionKey.OP_CONNECT （8）**
- **接收 : SelectionKey.OP_ACCEPT （16）**

若注册时不止监听一个事件，则可以使用“位或”操作符连接。
**int interestSet = SelectionKey.OP_READ|SelectionKey.OP_WRITE**

#### （4）io多路复用

根据操作系统的知识我们知道，计算机的硬件资源是有内核态来控制的，而我们的程序是运行在用户态中的，但是IO相关的函数（即`系统调用`）是运行这内核态的（这里给出一个Linux系统调用列表的连接：[https://blog.csdn.net/baobao8505/article/details/1115815](https://link.zhihu.com/?target=https%3A//blog.csdn.net/baobao8505/article/details/1115815)），所以当发生IO读写操作时，会有一个用户态到内核态的切换过程，这个过程是代价是比较大的，所以我们要想办法减少内核态与用户态的切换次数。

![img](https://pic3.zhimg.com/80/v2-6e6a99b975f1d34b120a2b6f3e1764b2_1440w.jpg)

NIO代码中，就存在这这一个问题，每当我们调用一次`read`时，都会产生一次用户态到内核态的切换，这就造成了很大的资源浪费，所以我们能不能一次性把要检查的Socket连接都丢给内核，然后让内核告诉我们有哪些Socket有客户端要来连接了，有哪些Socket有客户端给它发送数据了，然后再针对这些连接做相应的处理就行了，而不是一个一个Socket连接去找内核检查对应的状态？这个时候`IO多路复用器`就应运而生了。

##### 1 、select模型

select模型是一个跨平台的IO模型，在Windows和Linux系统中均有相应的实现。我们通过对调用select这一个系统调用来使用select模型。它的优缺点如下：
`优点`：跨平台支持。
`缺点`：
（1）每次调用select时都需要把fd集合（socket集合，fd为文件描述符缩写）从用户态拷贝到内核态，在fd集合较大时，系统开销比较⼤。
（2）内核仍然需要遍历fd集合，在有过多空闲socket连接时，仍会造成资源浪费。
（3）select所支持的fd数量太小了，默认只有1024。
select的函数定义I如下：

```text
int select(int maxfdp, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

它所需的参数分别是：`maxfdp`所有文件描述符中的最大值+1（在Windows中没有意义，只是为了兼容伯克利套接字规范），`readfds`是需要监听读事件的文件描述符集合，`writefds`是需要监听写事件的文件描述符集合，`exceptfds`是需要监听异常事件的文件描述符集合，`timeout`是超时时间。
它的大概工作流程如下：

![img](https://pic4.zhimg.com/80/v2-152b26089309e87a03e5a582b78d00c3_1440w.jpg)

##### 2、 poll模型

poll模型是对select模型的改进，但是Windows并没有poll的相应实现，Linux中才有相应的实现。poll改进了select最大fd数量的限制，但依然没有解决掉select的需要遍历fd集合问题和内存拷贝问题。他的优缺点如下：
`优点`：
（1）不要求调用者计算所有fd的最大值+1的大小。
（2）简化了select三种集合操作的流程。
（3）没有fd数量限制。
`缺点`：
（1）不能跨平台。
（2）select模型缺点中的（1）和（2）。
pll的函数定义I如下：

```text
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

他所需的参数分别是：`fds`指向结构体数组的第0个元素的指针，`nfds`是用来指定fds数组元素的个数，`timeout`是超时时间。
pollfd结构体定义如下：

```text
struct pollfd{
	int fd;			
	short events;	
	short revents;	
};
```

它的大概工作流程如下：

![img](https://pic4.zhimg.com/80/v2-84991f9fc582babb9f978fc9cb2581bf_1440w.jpg)

##### 3、 epoll模型

epoll模型是对select模型和poll模型的超级增强版本，他也只在Linux系统的有对应的实现，在Windows下并没有对应的实现。而我们所用的Redis和Nginx就是使用了epoll模型，这也是为什么Redis没有Windows版本的原因。epoll的优缺点如下：

`优点`：
（1）epoll则没有对描述符数目的限制，它所支持的文件描述符上限是整个系统最大可以打开的文件数目，例如，在1GB内存的机器上，这个限制大概为10万左右。
（2）IO效率不会随d的增加而线性下降，只有活跃可用的fd才会调用callback函数。

> 看到有的文章说epoll使用了mmap技术来实现fd在内核态与用户态之间的零拷贝问题，但是有某些看了Linux源码的大神说并没有使用该技术，但是博主并没有看过对应的源码，无法验证哪个观点正确，所以这点暂时存疑。想了解mmap技术的请移步：https://zhuanlan.zhihu.com/p/83398714

`缺点`：调用相对繁琐，对程序员的能力要求较高。
epoll是提供了三个函数来供程序员调用，这三个函数是配套的，分别是`epoll_create、epoll_ctl、epoll_wait`，它们的定义如下：
epoll_create：

```text
int epoll_create(int size);
```

这个函数是用来创建epoll的，它所需的参数是：`size`表示要监听事件列表的长度的初始值，后面会根据实际情况自动调整。
epoll_ctl：

```text
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

这个函数是用来管理所创建的epoll中的fd的（增删改），它所需的参数是：`epfd`表示`epoll_create`函数的返回值，即epoll对应的fd，`op`表示要进行的操作（用三个宏来表示，分别是：`EPOLL_CTL_ADD`：注册新的fd到epfd中；`EPOLL_CTL_MOD`：修改已经注册的fd的监听事件;`EPOLL_CTL_DEL`：从epfd中删除一个fd），`fd`表示要监听的fd，这里必须是支持NIO的fd（比如socket），event表示对要监听事件的描述。
epoll_event 的定义如下：

```text
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t events; 
    epoll_data_t data; 
};
```

epoll_wait：

```text
int epoll_wait(int epfd, struct epoll_event * evlist, int maxevents, int timeout);
```

这个函数是用来等待epoll中的事件发生的，它所需要的参数是：`epfd`表示`epoll_create`函数的返回值，即epoll对应的fd，`evlist`是`epoll_wait`函数的返回数组，里面会存放着有事件发生的fd，`maxevents`表示`evlist`数组的大小，`timeout`是超时时间。

> 这里只是简单介绍一下epoll对应的函数及其工作流程，epoll的工作模式（LT和EL）其实也是我们需要关注的问题，但是由于篇幅问题，这里不再赘述，感兴趣的可以参考这篇博客：[https://www.cnblogs.com/xuewangkai/p/11158576.html](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/xuewangkai/p/11158576.html)

它的大概工作流程如下：

![img](https://pic2.zhimg.com/80/v2-dbb29543d439a07c0d823d419b2733c9_1440w.jpg)