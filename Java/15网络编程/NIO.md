# 笔记

## 操作系统底层的多路复用与Java中的多路复用

操作系统：

<img src="./image/多路复用01.png" width=500 height=500>

Java：

<img src="./image/多路复用02.png" width=500 height=500>

## Java构建多路复用的请求模型

阻塞过程主要发生在两个阶段上：

- 第一阶段：等待数据就绪；
- 第二阶段：将已就绪的数据从内核缓冲区拷贝到用户空间；

这里产生了一个从内核到用户空间的拷贝，主要是为了系统的性能优化考虑。假设，从网卡读到的数据直接返回给用户空间，那势必会造成频繁的系统中断，因为从网卡读到的数据不一定是完整的，可能断断续续的过来。通过内核缓冲区作为缓冲，等待缓冲区有足够的数据，或者读取完结后，进行一次的系统中断，将数据返回给用户，这样就能避免频繁的中断产生。

一个线程可以同时发起多个I/O调用，并且不需要同步等待数据就绪。在数据就绪完成的时候，**会以事件的机制**，来通知我们。

NIO中Selector是对底层操作系统实现的一个抽象，管理通道状态其实都是底层系统实现的。

毕竟JVM是一个屏蔽底层实现的平台，我们面向Selector编程就可以了。

```java
public class NioNonBlockingHttpClient {
    private static Selector selector;
    private static final Charset CHARSET = StandardCharsets.UTF_8;

    static {
        try {
            selector = Selector.open();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws IOException {
        NioNonBlockingHttpClient client = new NioNonBlockingHttpClient();
        for (String host : HttpConstant.HOSTS) {
            // 为每一个I/O调用都创建了一个SocketChannel，并关联到已经预设的Selector:
            client.request(host, HttpConstant.PORT);
        }
        // 关联完成后开启选择器监听:
        client.select();
    }

    public void request(String host, int port) throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.socket().setSoTimeout(5000);
        SocketAddress remote = new InetSocketAddress(host, port);
        // 此语句必须在connect之前设置才能开启SocketChannel的非阻塞模式:
        socketChannel.configureBlocking(false);
        socketChannel.connect(remote);
        // 关联channel和选择器，监听每个channel的连接就绪事件、读就绪事件、写就绪事件:
        socketChannel.register(selector,
                SelectionKey.OP_CONNECT
                        | SelectionKey.OP_READ
                        | SelectionKey.OP_WRITE);
    }

    // JavaNIO的Selector和Linux的select不同，这里不会遍历所有关联的channel，
    // 每次从选择器中获取的都是那些符合事件类型，并且完成就绪操作的channel，减少了大量无效的遍历操作。
    public void select() throws IOException {
        // 获取就绪的channel个数，
        // select()方法是同步阻塞的，直到有新的事件发生才会被唤醒，防止CPU空转的产生，
        // 也可以设置超时事件结束阻塞过程:
        while (selector.select(500) > 0) {
            // 获取channel中的就绪事件句柄key:
            Set keys = selector.selectedKeys();
            Iterator it = keys.iterator();
            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();
                it.remove();
                // 分别处理关心的事件，注意处理事件本身是阻塞过程，我们只是将等待就绪的过程变成了非阻塞:
                if (key.isConnectable()) {
                    connect(key);
                } else if (key.isWritable()) {
                    write(key);
                } else if (key.isReadable()) {
                    // 读就绪事件可能发生多次，这取决于处理方法中的缓存区大小
                    // 缓存区过大，只需一次读就绪事件即可将数据读取完毕。
                    // 缓存区过小，需要多次读就绪事件来完成数据读取。
                    receive(key);
                }
            }
        }
    }

    private void connect(SelectionKey key) throws IOException {
        System.out.println("连接就绪事件发生");
        SocketChannel channel = (SocketChannel) key.channel();
        channel.finishConnect();
        InetSocketAddress remote = (InetSocketAddress) channel.socket().getRemoteSocketAddress();
        String host = remote.getHostName();
        int port = remote.getPort();
        System.out.println(String.format("访问地址: %s:%s 连接成功!", host, port));
    }

    private void write(SelectionKey key) throws IOException {
        System.out.println("写就绪事件发生");
        SocketChannel channel = (SocketChannel) key.channel();
        InetSocketAddress remote = (InetSocketAddress) channel.socket().getRemoteSocketAddress();
        String host = remote.getHostName();
        String request = HttpUtil.compositeRequest(host);
        System.out.println(request);
        // 为什么不用封装的PrintWriter的write来向socket的输出流中写入数据？
        // 因为这个方法的阻塞过程是包含用户空间到内核缓冲区，以及等待socket真正将数据发送出去后才返回。
        // 我们这里要的仅仅是数据从用户空间拷贝到内核空间缓冲区这一步是阻塞的。
        // 至于OS将缓冲区的数据借由socket发送出去，这一步不应该在阻塞范围内。
        channel.write(CHARSET.encode(request));
        // 当写操作发生后，修改channel关心的事件，之后只关心读就绪事件发生。
        key.interestOps(SelectionKey.OP_READ);
    }

    private void receive(SelectionKey key) throws IOException {
        System.out.println("读就绪事件发生");
        SocketChannel channel = (SocketChannel) key.channel();
        // 不像BufferedReader的readLine方法自动帮我们从内核缓冲区读取数据到用户空间，
        // 在这里我们需要自己实现从内核缓冲区读取数据的操作，先分配一个固定大小的缓冲区，
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        // 从内核缓冲区中，拷贝数据到刚刚分配的固定大小的缓冲区，现在是往固定大小缓冲区中写入数据，属于写模式，
        channel.read(buffer);
        // 拷贝完毕后，需要转为读模式，此时会将position属性归零，表示下一次拷贝时可以从头开始放入，
        buffer.flip();
        // 从固定大小缓冲区中读取数据。
        String receiveData = CHARSET.decode(buffer).toString();
        if ("".equals(receiveData)) {
            // 没有数据可读时，取消channel与选择器的关联，关闭socket连接:
            key.cancel();
            channel.close();
            return;
        }
        System.out.println(receiveData);
    }
}
```
