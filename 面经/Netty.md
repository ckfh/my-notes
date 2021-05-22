# Netty 源码

## 启动流程要点

NIO 服务端的启动流程：

1. 创建 Selector；
2. 创建 ServerSocketChannel，设置为非阻塞模式；
3. 指定监听端口；
4. 将 SSC 注册到 Selector 上；
5. 关联感兴趣事件为连接事件。

Netty 服务端的启动流程要点（加粗的语句可以和上述启动流程对应起来）：

1. 因为 Netty 底层是 Java NIO，因此 Netty 服务端启动流程也就最终演化为 Java NIO 的服务端启动流程；
2. Netty 会在启动过程中将 NioServerSocketChannel 以 attachment 的形式绑定在 NIO 的 Selector；
3. 启动流程会涉及两个线程，一个是主线程，一个是 NIO 线程，启用 NIO 线程的目的是让 NIO 线程来负责启动流程中的注册和监听工作，使得主线程能以异步的方式继续向后运行；
4. **Selector 在服务端配置 bossEventLoopGroup 得到 bossEventLoop 就已经被创建好了**；
5. Netty 服务端启动主要分为了三个步骤：init、register、bind；
6. init() 方法在内部利用反射创建了 NioServerSocketChannel，**并将 ServerSocketChannel 也创建出来**，为 NSSC 添加了一个初始化 handler，初始化 handler 将在随后被执行，并且只会执行一次（主线程）；
7. register() 方法在内部由主线程负责将注册工作 register0 提交给 EventLoop(Boss) 线程池去执行，将注册工作的执行权交给 NIO 线程（主线程）；
8. register0() 方法内部调用了 doRegister() 方法，**将原生 SSC 注册到 EventLoop 所持有的 Selector 上**，并将 NSSC 作为附件和原生 SSC 进行绑定（NIO 线程）；
9. register0() 随后调用 invokeHandlerAddedIfNeeded() 方法，调用第 6 步中提到的 NSSC 上的初始化 handler 中的 initChannel() 方法，该方法的作用在于向 NSSC 注册一个 acceptor handler，在将来发生 accept 事件后建立连接（NIO 线程）；
10. register0() 随后再调用 safeSetSuccess() 方法向 promise 对象设置结果，触发 promise 对象上的回调方法的执行（NIO 线程）；
11. 在回调方法内部调用了 doBind0() 方法，该方法的作用在于将 bind() 方法的执行权交给 NIO 线程来执行，确保后续工作都由 NIO 线程来负责；
12. bind() 先调用了 doBind() 方法，**启动原生 SSC 的监听端口绑定操作**；
13. bind() 方法随后对上述经过一系列操作的原生 SSC 进行一个状态判断，如果判断成功，则表示原生 SSC 此时已经处于可用状态，触发 channel pipeline 上所有 handler 的 channelActive 方法执行，目前仅存在 head、acceptor、tail 这三个 handler，只有 head handler 中的 active 方法需要执行，**执行结果是为第 8 步中注册 Selector 返回的 selectionKey 添加 OP_ACCEPT 感兴趣事件**。

## **NioEventLoop** 要点

1. 由 Selector、Thread、任务队列、定时任务队列组成，Selector 为自身属性，Thread 和任务队列 taskQueue 从祖父类 SingleThreadEventExecutor 中继承过来，然后在其曾祖父类 AbstractScheduledEventExecutor 中又继承了定时任务队列 scheduledTaskQueue；
2. 任务队列是普通任务的队列，I/O 事件可以直接从 Selector 的 selectedKeys 上获取，不需要放到任务队列中；
3. NIO Selector 在 NEL 的构造方法当中通过调用 openSelector() 方法进行创建；
4. NEL 当中定义了两个 Selector 属性，一个叫 selector，另一个叫 unwrappedSelector，NIO Selector 被赋值给了后者，之所以要定义两个 Selector，是因为原生 NIO Selector 上的 selectedKeys 是基于 Set 实现，**Set 结构的遍历效率并不高**，底层借助 Map，需要先遍历桶数组，然后再遍历链表，因此 Netty  Selector 中的 selectedKeys 采用了基于**数组**的实现，提升遍历效率；
5. Netty 通过提供一个基于数组实现的 SelectedSelectionKeySet 利用反射替换掉了 NIO Selector 中基于 Set 实现的 selectedKeys 和 publicSelectedKeys，**为了在发生感兴趣 I/O 事件的集合遍历时能够提高效率**；
6. NEL Thread 只有在第一次调用 NEL 的 execute() 方法提交任务时才会被启动，属于懒加载机制，**并通过一个状态位来保证单线程**，线程启动之后，run() 方法内部通过一个**死循环**来不断地判断是否有任务可以执行；
7. 在 Thread 的死循环内部，为了防止无 I/O 事件或者无任务时的 CPU 空转现象，Netty 使用 selector.select(timeoutMillis) 进行阻塞，在超时时间到达，或者 I/O 事件发生，或者任务被提交时，都会唤醒 NIO 线程继续向下运行处理事件和任务，处理事件和处理任务对应的方法名称为 processSelectedKeys 和 runAllTasks；
8. 当其它线程向当前 NIO 线程提交任务时，会使用 wakeup() 方法尝试唤醒阻塞中的 NIO 线程，为了防止多个其它线程同时提交任务，频繁调用 wakeup() 方法（属于重量级操作），**使用一个变量来控制只有一个线程来负责唤醒**，后续线程就不需要再次唤醒。
9. 在每次死循环起始时，会尝试判断是否有任务可以执行，**如果当前没有任务则进入 SELECT 分支阻塞**，如果当前有任务，则顺带使用 selectNow() 尝试从 selector 上获取 I/O 事件，因为之后会跳过 SELECT 分支，进入处理事件和任务的流程，为的是尽可能地处理 I/O 事件；
10. NIO 空轮询 BUG（JDK 在 Linux 环境下的 selector 设计问题）发生时，即使没有事件发生，select() 方法也不会阻塞，这就导致 CPU 不断地进行空转，当出现该 BUG 的线程数量达到一定数量时，会导致 CPU 的使用率达到 100%；
11. Netty 解决 NIO 空轮询 BUG 的方法是使用一个整型计数器，判断在一定时间内，该计数器是否达到阈值 512，则判定发生了 BUG，此时 Netty 的做法是重建了一个新的 selector ，并将旧的 selector 上的事件转移到新的 selector 上；
12. 由于 NEL 内部的**单线程**不仅要执行 I/O 事件，还要执行其它任务，如果其它任务执行的时间过长，就会导致 I/O 事件的处理被延后，因此在 NEL 内部设置了一个 ioRatio 变量，用来控制处理 I/O 事件所占用的时间比例，默认为 50%，即一半时间处理 I/O 事件，一半时间处理其它任务，当执行普通任务的时间超过计算得到的执行时间时，便不会从队列中再次获取任务；
13. ioRatio 变量设置为 100 并不表示将所有时间花费在处理 I/O 事件上，相反，对于 100 会进入一个专门的判断分支，在该分支中，**对于普通任务的执行是不会指定执行时间的**，因此该变量设置为 <100 都比设置为 100 在优先处理 I/O 事件上更友好；
14. 在处理 I/O 事件的 processSelectedKeys 方法内部，默认会遍历数组形式的 selectedKeys，拿到发生事件的 SelectionKey，**通过 attachment() 方法获取 Key 上的附件**，这里的附件就是 NioServerSocketChannel，**因为接下来要对这个 Key 上的事件进行处理，需要从 NioServerSocketChannel 上获取 pipeline，然后获取每个 handler 来进行处理**；
15. 拿到发生事件的 key 和附件 NSSC 后，调用 processSelectedKey 方法，根据事件类型进行不同的处理。

## accept 流程

NIO 流程：

1. selector.select() 阻塞直到事件发生；
2. 遍历处理 selectedKeys；
3. 拿到一个 key，判断事件类型是否为 accept；
4. 创建 SocketChannel，设置为非阻塞；
5. 将 SocketChannel 注册到 selector；
6. 关注 selectionKey 的 read 事件。

accept 流程要点：

1. NIO 流程中的前三个流程已经在上一节的 NEL 流程中被走完，后三个流程则是根据不同的事件类型所作的不同处理；
2. 在 processSelectedKey 方法中，accept 和 read 事件都是调用的 unsafe.read() 方法（注意这里的 unsafe 对象会根据所在线程的不同而赋值不同的对象），在方法内部调用 doReadMessages() 方法，**通过原生 ServerSocketChannel 的 accept 方法得到原生 SocketChannel**，并将其作为参数构造 NioSocketChannel；
3. 将 NioSocketChannel 对象放到 readBuf 对象集合中，传递给 NioServerSocketChannel 的 pipeline，调用 pipeline 上的 handler 对这个 NioSocketChannel 对象进行处理；
4. 服务端的 acceptor handler 将 NioSocketChannel 的注册工作封装成一个 Runnable 对象，提交给 childEventLoopGroup 中的某个 childEventLoop 去执行，将注册工作的执行权进行转移；
5. 对于 NioSocketChannel 的 register0 过程，首先**调用 doRegister() 方法将原生 SocketChannel 注册到 childEventLoop 的 selector 上，并将 NioSocketChannel 作为附件进行绑定**；
6. register0 随后调用 invokeHandlerAddedIfNeeded() 方法执行 NioSocketChannel 上的初始化操作，即调用 NioSocketChannel pipeline 上的 handler，触发每个 handler 的 initChannel 方法（包括编写服务端启动器时为 NioSocketChannel 添加的 handler）；
7. 在 NioSocketChannel 的 head handler 中会调用 readIfIsAutoRead() 方法，该方法最终将调用 doBeginRead() 方法，**为 NioSocketChannel 的 selectionKey 上关注 read 事件**。

## read 流程

NIO 流程：

1. selector.select() 阻塞直到事件发生；
2. 遍历处理 selectedKeys；
3. 拿到一个 key，判断事件类型是否为 read；
4. 读取操作。

read 流程要点：

1. 当 NioSocketChannel 发生可读事件时，此时将构建一个池化的基于直接内存的 byteBuf，默认初始容量为 2048 个字节；
2. 随后将读取到的数据放到 byteBuf 当中，传递给 NioSocketChannel pipeline 上的 handler 逐个进行处理。

