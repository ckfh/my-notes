# 笔记

## 黑马Netty教程视频笔记

selector的出现有以下原因：

1. 多线程设计：内存占用高，上下文切换成本高，适合连接数少的场景。
2. 线程池版设计：socket类的操作都是阻塞模式，即使没有数据可读，线程也会阻塞，线程阻塞此时就无法处理其它有数据可读的socket，因此线程池版设计适合短连接场景，比如HTTP请求，请求一次，释放一次。

selector管理多个channel，可以注意到每个channel上发生的事件，一旦发生事件，立马调用后续的单线程进行处理，它适合连接数多，但流量低的场景，流量低是因为多个事件是串行地在单线程中执行，因此事件业务处理时间不能太长。

和线程池版的区别在于：线程池的线程会作无用等待，无法及时处理其它已经准备好的对象。而selector的线程全程处于活跃状态，处理一个接一个的事件。

判断数据的字节长度的前提是知道数据的编码格式。

bytebuffer分为两类：

- 一类是heapbytebuffer-Java堆内存，读写磁盘文件效率低，受到GC影响。
- 一类是directbytebuffer-直接内存，读写效率高(零拷贝)，不受GC影响，分配效率低(直接内存分配需要借助操作系统底层来实现)，需要手动进行内存释放，使用不当会造成内存泄漏。

channel.read(buf)这是从文件读取数据到缓冲区，不是读取缓冲区的数据到channel。将channel理解成文件流，从文件流read，放到缓冲区中。channel.write(buf)将缓冲区数据写入到文件。

用charset和wrap将字符串写入bytebuffer时，会自动切换为读模式，即将position置为零。

bytebuffer的操作是非线程安全的。

已知数据子内容长度，一种“分散读取”的思想是根据子内容长度创建多个bytebuffer，然后channel.read(buf数组)，一次性将文件内容读取到多个bytebuffer中。“集中写入”channel.write(buf数组)，一次性将多个bytebuffer写入文件。

黏包现象的原因是传输时为了效率，将多个消息组合在一起发送。

半包现象则是因为传输的大小有一定限制，消息被组合到前一个包时，剩余一部分超出了限制，因此将剩余部分留到下一个包进行发送，因此出现了半包。

channel有一个写入的上限，即channel.write(buf)不能保证一次性将buffer中的内容全部写入channel，因此需要配合while循环，判断buffer是否有剩余。

filechannel必须工作在阻塞模式下，并且它的写入需要调用force(true)来强制写入数据，这是因为操作系统出于性能考虑，会缓存数据，不是立刻写入磁盘。

JDK底层带transferTo关键字的方法底层都是借助操作系统的零拷贝机制进行传输优化。一次transferTo有2G数据量的最大限制，可以分多次进行传输。

从selectedKeys中拿SelectionKey进行处理时，要将其从selectedKeys删除。

- Selector：所有的key集合
- selectedKeys：发生事件的key集合

selectedKeys有个特点，如果我们不手动删除某个key，那么这个key会一直留在这个集合中，我们处理key上的事件只是将这个key对象的事件状态置为已处理，但是key对象本身仍然保留在集合中。

这个特点就会导致原本已经处理完事件的key因为排在集合中的较前位置，因此遍历处理key事件时，仍然会尝试对其上的事件进行处理，导致空指针异常。

例如连接key在前，连接建立后将读key加入，此时连接key事件处理完毕，之后读key事件发生，遍历时仍先遍历到连接key，此时调用accept()方法但实际上并没有连接到达，因此方法返回的channel为null，当对这个channel进行非阻塞配置时，报了空指针异常。

当selector中某个key上的关注事件发生时，会将这个key放到selectedKeys中，当这个事件被处理后，selectedKeys中该key的事件状态变为已完成，仍保留于selectedKeys当中，因此每次从selectedKeys取出key进行处理前或者处理后都需要将其从selectedKeys中移除，防止正常处理后的key对后续迭代造成影响，而想要移除Selector上的key，则需要借助key对象的cancel方法。

如果客户端直接中断与服务端的连接，会触发read事件，如果在处理read事件的逻辑内尝试读取内容，将抛出I/O异常，因为此时连接已经被关闭，这会导致服务端直接被中断，也就是说在这里我们应该抓住这个I/O异常并进行处理，因为客户端连接已经断开，因此该客户端key已经没有必要进行关注，使用cancel方法将其从Selector的key集合中移除，这样就不会造成外层的不断循环，因为我们并没有处理这个key上的事件(想读却读不了)，因此下一次selector又将其放入到selectedKeys中，又进行了一次迭代，不管是正常断开还是强制断开，都会触发read事件，如果是正确断开，read方法返回-1，强制断开read方法会抛出I/O异常。

通过关联附件的方式将bytebuffer与SelectionKey进行关联，两者的生命周期是一致的。

- connect-客户端连接成功时触发
- accept-服务端成功接受连接时触发
- read-数据可读入时触发，有因为接收能力弱，数据暂不能读入的情况
- write-数据可写出时触发，有因为发送能力弱(比如操作系统进行网络发送时的缓冲区大小限制)，数据暂不能写出的情况，意思就是数据不一定每时每刻都可以从buffer写入到channel中，有可能出现写入了部分数据后，无法再次写入的情况，需要多次调用write函数不断写入。

单线程NIO的弊端：1.单线程无法充分利用多核CPU的性能。2.单线程如果遇到耗时的事件会导致其它就绪事件被耽误。

多线程优化的一个思路：一个线程持有一个selector，其中一个selector叫做boss，将serversocketchannel注册到该selector上，它负责在服务端处理accept事件，然后将得到的socketchannel对象注册给另一个线程中的一个名叫worker的selector，它负责处理这个channel的read事件。

CPU密集型选择的worker数可以是CPU的核心数；IO密集型选择的worker数可以按照一个效率公式来得到。

调用selector.wakeup()会让阻塞的selector被唤醒，其原理类似locksupport的park和unpark的发门票原理。

stream vs channel：

- stream不会自动缓冲数据，channel会利用系统提供的发送缓冲区、接收缓冲区（更为底层）
- stream仅支持阻塞API，channel同时支持阻塞、非阻塞API，网络channel可配合selector实现多路复用
- (网络编程中)二者均为全双工，即读写可以同时进行

- 同步：线程自己获取结果(调用方法的线程和接收数据处理结果的线程是同一个)
- 异步：线程自己不去获取结果，而是由其它线程送结果(至少两个线程)
- netty异步：使用多线程将方法调用和处理结果相分离，当前线程在方法调用后，将处理结果的过程交给另一个线程来处理，解放了当前线程

NioEventLoopGroup里面包含了多个NioEventLoop，而每个NioEventLoop又包含了一个selector对象和一个thread对象，其实就类似于上述提到的boss和worker，由selector监听channel上的事件，交给thread对象来执行。

实际上NioEventLoop的“单线程”是一个单线程线程池，因此NioEventLoop除了能处理I/O的读写任务外，还可以向它提交普通任务或者定时任务。

NioEventLoop能处理IO事件，普通任务，定时任务；DefaultEvent能处理普通任务，定时任务。

它俩可以联合起来实现异步，例如NioEventLoop从IO事件中读取到数据，交给DefaultEvent进行业务处理，于是NioEventLoop就可以继续去处理其它的IO读写事件，然后DefaultEvent完成业务处理后，将数据交给NioEventLoop返回给客户端，或者NioEventLoop本身就可以联合另一个NioEventLoop来进行异步处理。

从网络输入流读取数据=》业务处理=》将结果写入到网络输入流中。

channel就是数据传输的通道，msg理解为流动的数据，最开始输入是ByteBuf，但经过pipeline的加工，会转为其它类型对象，最后输出又变成ByteBuf。

NioEventLoopGroup如果没有指定初始容量，则默认是CPU核心数乘以2。

定时任务的一个用处是可以在长连接(keepalive)时对连接进行保活处理。

EventLoop利用pipeline中多个handler对数据进行加工处理，正常来说多个handler的执行过程都由同一个EventLoop中的线程来执行，但实际上可以选择将部分handler的执行权交给其它的EventLoop来帮忙执行，异步化业务处理。

ChannelHandlerContext：使一个ChannelHandler能够与它的ChannelPipeline和其它处理程序进行交互。其中，处理程序可以通知ChannelPipeline中的下一个ChannelHandler，也可以动态地修改它所属的ChannelPipeline。

如果两个handler绑定的是同一个EventLoop，那么就可以连续调用，否则，把要调用的代码封装为一个runnable对象，进行线程切换，由下一个handler的EventLoop来调用。

ChannelFuture可以联想到JDK的Future对象，一般是配合异步方法使用的，用来正确处理异步方法的结果。

- channelFuture.sync(); // 阻塞直到连接建立
- future.get(); // 阻塞直到方法执行完毕
- channelFuture.addListener(回调对象); // 将连接建立后的处理操作都交给 NIO 线程进行处理

- 同步：谁发起请求谁处理结果
- 异步：只发起请求，等结果和处理结果都交给另外线程

- channelFuture是用来异步处理连接建立的；
- closeFuture是用来异步处理连接关闭的；
- 可以知道netty中其实很多方法都是异步的，因此提供了对应的future对象来帮助正确处理运行逻辑。

- netty的异步并没有缩短响应时间，反而有所增加，这是因为单个任务被拆分，多个子任务协同时需要同步。
- netty的异步增加的是任务的吞吐量。
- 合理的进行任务拆分，是利用异步的关键。

future/promise可以理解为就是两个线程之间传递消息的一个容器，一个线程执行后将结果放到容器中，另一个线程通过容器获取结果。

JDK和NETTY的future特点是提交任务后才返回的一个对象，future的创建权和对结果的设置权都是被动的。

promise对象可以不需要等待任务结束，在任务执行过程中，可以直接给任务的结果设置成功或者失败，是脱离了任务独立存在的。

一个pipeline的handler调用链如下：head->h1->h2->h3->h4->h5->h6->tail，假设其中123是入站处理器，456是出站处理器，入站处理器的调用顺序为123，出站处理器的调用顺序为654，并且只有在服务端写出数据的时候才会触发出站处理器的调用流程。

源码中的fireChannelRead(invokeChannelRead)就是调用pipeline上的所有handler的channelRead方法(入站处理器方法)。

super.channelRead(ctx, msg);ctx.fireChannelRead(msg);都可以用来将数据传递给下一个handler。

ctx.writeAndFlush，从当前处理器开始往前找出站处理器；channel.writeAndFlush，从tail开始往前找出站处理器。

堆内存分配效率高，I/O效率低；直接内存分配效率低，I/O效率高；

netty的bytebuf相较于nio的bytebuffer只用一个指针操控读写，它同时提供了读指针和写指针来控制可读和可写的区域，并且拥有自动扩容机制。

默认创建的bytebuf是基于池化的直接内存上的bytebuf对象，池化是为了复用这些bytebuf，直接内存是为了提高与硬件的I/O能力。

因为默认实现是直接内存，因此直接内存的回收最好是手动进行回收，而不是等待操作系统本身的垃圾回收。

- UnpooledHeapByteBuf 使用的是 JVM 内存，只需等 GC 回收内存即可
- UnpooledDirectByteBuf 使用的就是直接内存了，需要特殊的方法来回收内存
- PooledByteBuf 和它的子类使用了池化机制，需要更复杂的规则来回收内存

- Netty 这里采用了引用计数法来控制回收内存，每个 ByteBuf 都实现了 ReferenceCounted 接口
- 每个 ByteBuf 对象的初始计数为 1
- 调用 release 方法计数减 1，如果计数为 0，ByteBuf 内存被回收
- 调用 retain 方法计数加 1，表示调用者没用完之前，其它 handler 即使调用了 release 也不会造成回收
- 当计数为 0 时，底层内存会被回收，这时即使 ByteBuf 对象还在，其各个方法均无法正常使用

注意：并不是使用了ByteBuf就会使其计数加一，因为整个数据流中ByteBuf就存在一份。(emm)可以这样理解，bytebuf对象初始计数为1，然后在handler方法内部我们用局部变量对其进行引用，此时引用计数会加一，但是在方法调用结束后局部变量被清除，此时对象的计数会减一，因此如果方法内部已经进行过减一操作，再加上最后方法出栈时的减一，就可以成功对象的引用计数置为零，触发内存回收。

pipeline中哪个handler最后使用了ByteBuf，就由它来负责释放ByteBuf。

上述提到的head和tail这两个内置handler，会在出站和入站的最后进行ByteBuf的释放工作。但是！如果中间的handler将ByteBuf转换成了其它格式并向后传递，那么最后head和tail拿到的就不一定是ByteBuf本身，就不会进行释放工作。因此不要依赖head和tail来进行ByteBuf的内存释放工作。

在TailContext的channelRead方法内部调用了ReferenceCountUtil这个工具类，它会判断当前对象是否为ByteBuf类型，如果是就强转然后调用release方法。在HeadContext的write方法内部同样调用了ReferenceCountUtil工具类来对ByteBuf对象的内存进行释放。

有关“谁是最后使用者，谁负责release”的分析如下：

1. 起点，对于 NIO 实现来讲，在 io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read 方法中首次创建 ByteBuf 放入 pipeline（line 163 pipeline.fireChannelRead(byteBuf)）
2. 入站 ByteBuf 处理原则
    1. 对原始 ByteBuf 不做处理，调用 ctx.fireChannelRead(msg) 向后传递，这时无须 release
    2. 将原始 ByteBuf 转换为其它类型的 Java 对象进行传递时，这时 ByteBuf 就没用了，必须 release
    3. 如果不调用 ctx.fireChannelRead(msg) 向后传递，那么也必须 release
    4. 注意各种异常，如果 ByteBuf 没有成功传递到下一个 ChannelHandler，必须 release
    5. 假设消息一直向后传，那么 TailContext 会负责释放未处理消息（原始的 ByteBuf）
3. 出站 ByteBuf 处理原则
    1. 出站消息最终都会转为 ByteBuf 输出，一直向前传，由 HeadContext flush 后 release
4. 异常处理原则
    1. 有时候不清楚 ByteBuf 被引用了多少次，但又必须彻底释放，可以循环调用 release 直到返回 true

**操作系统的零拷贝是指数据从磁盘读入内核缓冲区，再由缓冲区不经过Java内存，不经过socket缓冲区，直接发往网卡驱动，这是目前对于零拷贝的最高级优化，减少多次的数据复制操作**。

**netty零拷贝的概念也是减少数据复制的次数，但不是操作系统层面的那种零拷贝**。

slice：

对一个ByteBuf进行逻辑上的切片，每个切片维护独立的读写指针，但实际上它们都在同一块ByteBuf内存区域上进行操作，只是操作的区域不一样。并不会创建新的ByteBuf，然后将数据拷贝过去。

如果进行了切片操作，最好跟随一次retain操作，防止原始的ByteBuf调用release释放内存后，影响当前slice的读写，如果当前切片操作结束，再自己手动调用一次release方法释放内存。

对原始ByteBuf的读写不会影响slice的读写指针，因此双方用的是两套独立的读写指针。

但是对原始ByteBuf的更改会影响到slice的内容，因为双方共用同一块内存。

duplicate:

可以理解为是slice的整体切片版。

copy(真正意义上的拷贝):

slice和duplicate都是netty零拷贝的体现之一，因为并没有创建新的ByteBuf对象，所以减少了数据复制的操作。

CompositeByteBuf是在逻辑上将多个ByteBuf连接在一起的对象，减少内存复制，缺点是复杂，多次操作带来性能的损耗。

Unpooled是一个工具类，它提供了一个wrappedBuffer方法，可以传入buf或者字节数组，将它们组成一个新的buf对象，它的底层调用的就是CompositeByteBuf，防止数据拷贝操作。

ByteBuf 优势

- 池化 - 可以重用池中 ByteBuf 实例，更节约内存，减少内存溢出的可能
- 读写指针分离，不需要像 ByteBuffer 一样切换读写模式
- 可以自动扩容
- 支持链式调用，使用更流畅
- 很多地方体现零拷贝，例如 slice、duplicate、CompositeByteBuf

有关读和写的一个误解：

我最初在认识上有这样的误区，认为只有在 netty，nio 这样的多路复用 IO 模型时，读写才不会相互阻塞，才可以实现高效的双向通信，但实际上，Java Socket 是全双工的：在任意时刻，线路上存在`A 到 B` 和 `B 到 A` 的双向信号传输。即使是阻塞 IO，读和写是可以同时进行的，只要分别采用读线程和写线程即可，读不会阻塞写、写也不会阻塞读。就是说socket通信是建立两条通道，一条通道负责接受数据，另一条通道负责发送数据。

关于粘包和半包的再理解：

这是因为TCP协议本身是一种流式协议，消息是无边界的，我们必须自己定义和识别边界，因此上层协议必须自己定义和识别边界。

粘包现象：发送abc和def，接收abcdef。

原因：

- 应用层：接收方的buffer设置太大；
- 传输层：TCP协议定义了滑动窗口(缓冲区、流量控制)，假设接收方处理不及时且窗口大小足够大，那么一个完整报文就会缓冲在接收方的滑动窗口中，当滑动窗口缓冲了多个报文就会粘包；
- 传输层：nagle算法，报文是由首部和数据部分组成的，有的时候首部字节很多，数据部分字节很少，nagle算法就会尝试缓存，直到有足够数据了才发送。

半包现象：发送abcdef，接收abc和def

原因：

- 应用层：接收方的buffer小于实际到达的数据量；
- 传输层：滑动窗口，接收方只能接收报文的一半，因此只能先发送一半，此时滑动窗口满了，等待处理回复ACK后，再发送另一半；
- 链路层：MSS限制，因为承载能力有限，数据会被切分发送。

黏包和半包的解决方案

1. 短连接：发一个包建立一次连接，这样连接建立到连接断开之间就是消息的边界，效率太低。
2. 每一条消息采用固定长度，缺点浪费空间。
3. 每一条消息采用分隔符，例如 \n，缺点需要转义
4. 每一条消息分为 head 和 body，head 中包含 body 的长度

TCP/IP 中消息传输基于流的方式，没有边界。

协议的目的就是划定消息的边界，制定通信双方要共同遵守的通信规则。

如何设计协议呢？其实就是给网络传输的信息加上“标点符号”。但通过分隔符来断句不是很好，因为分隔符本身如果用于传输，那么必须加以区分。因此，下面一种协议较为常用：定长字节表示内容长度 + 实际内容。

**入站时对bytebuf进行decoder转为相应类型，出站时对相应类型进行encoder转为bytebuf**。

`SimpleChannelInboundHandler<Object>`，表示该handler只对上一个handler传递过来的Object类型的数据感兴趣，其它类型的数据则会直接跳过，例如`SimpleChannelInboundHandler<HttpRequest>`表示只对传递过来的HttpRequest对象感兴趣。

HTTP作为一种常见的web服务端和客户端之间的通信协议，netty底层已经内置了相关的编解码处理器。

自定义协议

- 魔数，用来在第一时间判定是否是无效数据包
- 版本号，可以支持协议的升级
- 序列化算法，消息正文到底采用哪种序列化反序列化方式，可以由此扩展，例如：json、protobuf、hessian、jdk
- 指令类型，是登录、注册、单聊、群聊... 跟业务相关
- 请求序号，为了双工通信，提供异步能力
- 正文长度
- 消息正文

我们说的协议通常是指应用层的协议，为什么我们不用HTTP协议来进行开发而是选择指定自定义协议呢？

根据应用的不同，服务器不总是web服务端，例如redis服务端就采用redis协议。

**就是一句话，一种协议针对的是一种场景，HTTP协议的格式或者字段对于一些非web应用的场景来说就属于多余或没用**。

handler如果记录多个线程之间保存的状态，那它就是线程不安全的，不应该被作为共享实例。

比如LengthFieldBasedFrameDecoder它对于未达到指定长度的消息会暂存，因此不应该在多个eventloop之间共享。

对于有@Sharable注释的handler来说，它可以提取作为一个共享变量，在多个eventloop之间共享，例如日志handler，它不保存状态，有多少数据就都打印出来。

如果一个handler可能保存**上一次处理**的状态，那它就不能使用@Sharable注解，不能被多个eventloop中的线程共享。

ByteToMessageCodec子类不允许设置@Sharable注解是因为netty认为该类型是可能被设计成会暂存一次数据状态的，因此netty为了防止线程不安全，因此强制其不能使用@Sharable注解。

对于自定义的codec类如果想要该注解可以实现MessageToMessageCodec。

学习netty框架时，对于数据的展示总是直接在channelRead方法内部直接打印，如果想要将其带出当前这个eventloop，可以利用channel自身提供的一个类似哈希映射的存储方式，`ctx.channel().attr(key).set(msg);`将消息存放在channel中，这样我们就可以在能获取到channel对象的地方再通过相同的key将消息获取出来。

@Sharable:Indicates that the same instance of the annotated ChannelHandler can be added to one or more ChannelPipelines multiple times without a race condition.
If this annotation is not specified, you have to create a new handler instance every time you add it to a pipeline because it has unshared state such as member variables.

指示可以将带注释的ChannelHandler的相同实例多次添加到一个或多个channelpipeline，而不需要竞争条件。

如果未指定此注释，则每次将其添加到管道时都必须创建一个新的处理程序实例，因为它具有非共享状态(如成员变量)。

！！！**所以在添加handler的时候，如果handler被定义成一个独立的变量在多处引用，它必须注解上@Sharable**，否则将导致多个channel竞争该handler，导致只有一个channel能够拥有这个handler的使用权，后续channel连接时将无法连接。

这告诉我们对于拥有共享状态的handler，必须为每个channel都准备，否则多个channel会发生竞争。

连接假死

1. 网络设备出现故障，例如网卡，机房等，底层的 TCP 连接已经断开了，但应用程序没有感知到，仍然占用着资源。
2. 公网网络不稳定，出现丢包。如果连续出现丢包，这时现象就是客户端数据发不出去，服务端也一直收不到数据，就这么一直耗着。
3. 应用程序线程阻塞，无法进行数据读写。

如果能收到客户端数据，说明没有假死。因此策略就可以定为，每隔一段时间就检查这段时间内是否接收到客户端数据，没有就可以判定为连接假死。

定时心跳

客户端可以定时向服务器端发送数据，只要这个时间间隔小于服务器定义的空闲检测的时间间隔，那么就能防止前面提到的误判。所谓误判就是当前客户端只是离开状态，而不是网络设备出问题或公网网络不稳定或线程阻塞。

半连接和全连接队列

第一次客户端握手到达服务器端，此时会将连接对象放到半连接队列，等待三次握手完毕，会将连接对象从半连接队列放到全连接队列。

为什么建立好的连接不直接使用，而是再转移到全连接队列，因为服务端accept()方法能力有限，三次握手是发生在accept()方法之前的。

如果全连接队列满载，服务端将直接发送一个拒绝连接的错误消息到客户端。

注意：队列满载是在accept()方法无法处理的时候，但是netty本身的accept()方法处理能力是非常强的。

Windows backlog参数可以直接控制全连接队列大小，Linux则需要配合修改底层配置文件。

如何在源码中找到一个变量的起始配置地点？

1. 思考该变量在哪里被传入。
2. 查看方法在哪里被调用。
3. 找到具体实现。

例如 backlog 参数

1. 它在ServerSocketChannel 的bind(SocketAddress local, int backlog)方法被传入。
2. 右键bind方法“find usage”找到在源码其它位置的调用处。NioServerSocketChannel的doBind方法中"javaChannel().bind(localAddress, config.getBacklog());"。
3. config是ServerSocketChannelConfig类型变量，找到它的一个默认实现DefaultServerSocketChannelConfig。
4. private volatile int backlog = NetUtil.SOMAXCONN;
5. 找到SOMAXCONN在NetUtil中的赋值语句。

结论：Windows默认全连接队列大小200，Linux默认4096(5.8.0-48-generic)，但是Linux内核会根据传入的backlog参数和底层配置文件的somaxconn参数，取两者的最小值。

ulimit -n：操作系统参数，用于调整每个进程可打开的最大文件描述符数量，为了支持高并发，服务端肯定需要调整该参数的值。该参数属于临时修改，因此最好配置在启动脚本中。

TCP_NODELAY：socketchannel参数，可以避免nagle算法对包的缓存，建议设为true。

**不要自己配置滑动窗口的大小，因为现在操作系统已经可以自适应地调整窗口大小，如果自己设置反而画蛇添足**。

要学会看源码知道如何配置一个参数的值，比如ctx.alloc()方法分配buf默认是池化的直接内存，如果改成非池化的堆内存呢？通过跟踪源码知道在虚拟机启动时设置io.netty.allocator.type=unpooled和设置io.netty.noPreferDirect=true。

netty对于用于I/O操作的buf强制使用直接内存，因为直接内存和I/O打交道效率就是高。但池化和非池化还是可以选择的。

而我们自己在handler内部分配的buf既可以是堆内存也可以是直接内存。

netty接收缓冲区使用的buf就属于和I/O操作打交道的直接内存。

日志处理器可以用来调试消息的出站和入站流程，假如日志处理器位于编解码处理器的前面，出站时没有打印日志，那么多半就是编码出了问题。

writeAndFlush也是异步方法，对其调试可以使用其返回的future对象，获取writeAndFlush的返回结果。

promise就是一个容器，它可以在多个线程之间交换结果。

@Sharable就表示当前处理器内部不应该存在所谓的共享变量(状态保存)，否则会存在线程安全问题，**但是一方面，@Sharable只是提醒你自己要去处理线程安全问题，如果我们自己考虑线程安全问题，那就可以使用@Sharable注解**。

ChannelHandler的事件回调方法中，ctx.close() 或 ctx.channel().close()只能关闭与某个客户端连接的channel，而ctx.channel().parent()获取到的是ServerChannel，所以调用ctx.channel().parent().close()才能关闭server，使server启动线程从阻塞状态恢复，从而在finally代码块中执行退出关闭操作。也可以这样理解，服务端我们设置SocketChannel处理器时用的是childHandler，从child联想到parent。

NIO启动主线分析：

1. 创建一个selector。
2. 创建一个SSC(serversocketchannel)。
3. 将SSC注册到selector拿到selectionkey。
4. SSC绑定监听端口。
5. 利用selectionkey设置selector对SSC的感兴趣事件。

netty启动主线就是围绕上述步骤进行的：第1个步骤在NioEventLoopGroup实例化时被创建。剩下的步骤都是在bind方法中实现的。

NioEventLoop由selector、线程、任务队列组成。

NioEventLoop除了处理I/O事件，还可以处理普通任务和定时任务。

NioEventLoop提供两个selector对象的目的是其中一个负责引用原生的selector对象，由于原生selector的selectionkeys集合是使用set数据结构对key进行存储的，其遍历性能并不高(哈希表，遇到链表或者树还得在链表和树上进行遍历)，因此netty将其替换成了数组实现。

**但实际上你说性能提升得明不明显，其实也不明显，但是netty在很多地方都有类似的优化实现，积少成多，整体的性能提升就非常明显了**。

因此NioEventLoop维护的另一个selector对象就是对原生selector对象的封装。保留原生selector对象是为了调用它的方法，只是在遍历key集合时用的是基于数组实现的集合。

selector.wakeup()是一个重量级操作，阻塞到运行，要借助系统调用，借助系统调用意味着用户态到核心态的切换。

注意selector.select()只会响应I/O事件的唤醒，如果是普通任务想要唤醒则需要调用wakeup()方法。

**任务队列是普通任务和定时任务的队列**。I/O事件可以通过selector的keys集合获取。

NIO空轮询BUG，这是在Linux底层使用epoll作为系统调用时会发生的一个BUG，指的是没有IO事件发生，但是select方法被唤醒，又因为select和后续的方法处理逻辑通常放在一个while(true)当中，这就会导致CPU空转，利用率达到100%。
