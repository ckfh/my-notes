# 网络编程

## TCP编程

使用Socket进行网络编程时，本质上就是两个进程之间的网络通信。其中一个进程必须充当服务器端，它会主动监听某个指定的端口，另一个进程必须充当客户端，它必须主动连接服务器的IP地址和指定端口，如果连接成功，服务器端和客户端就成功地建立了一个TCP连接，双方后续就可以随时发送和接收数据。

因此，当Socket连接成功地在服务器端和客户端之间建立后：

- 对服务器端来说，它的Socket是指定的IP地址和指定的端口号；
- 对客户端来说，它的Socket是它所在计算机的IP地址和一个由操作系统分配的随机端口号。

### 服务器端

注意到代码ss.accept()表示每当有新的客户端连接进来后，就返回一个Socket实例，这个Socket实例就是用来和刚连接的客户端进行通信的。由于客户端很多，要实现并发处理，我们就必须为每个新的Socket创建一个新线程来处理，这样，主线程的作用就是接收新的连接，每当收到新连接后，就创建一个新线程进行处理。

这里也完全可以利用线程池来处理客户端连接，能大大提高运行效率。

如果没有客户端连接进来，accept()方法会阻塞并一直等待。如果有多个客户端同时连接进来，ServerSocket会把连接扔到队列里，然后一个一个处理。对于Java程序而言，只需要通过循环不断调用accept()就可以获取新的连接。

```Java
public class Server {
    public static void main(String[] args) throws IOException {
        // 没有指定IP地址，表示在计算机的所有网络接口上监听指定端口6666：
        ServerSocket ss = new ServerSocket(6666);
        System.out.println("server is running...");
        // 如果端口监听成功，则使用一个无限循环来处理客户端的连接：
        for (; ; ) {
            Socket sock = ss.accept();
            System.out.println("connected from " + sock.getRemoteSocketAddress());
            Thread t = new Handler(sock);
            t.start();
        }
    }
}

class Handler extends Thread {
    Socket sock;

    public Handler(Socket sock) {
        this.sock = sock;
    }

    @Override
    public void run() {
        // 通过Socket获取网络输入输出流：
        try (InputStream input = this.sock.getInputStream()) {
            try (OutputStream output = this.sock.getOutputStream()) {
                this.handle(input, output);
            }
        } catch (Exception e) {
            try {
                // 关闭Socket的同时也会关闭输入输出流：
                this.sock.close();
            } catch (IOException ioe) {

            }
            System.out.println("client disconnected.");
        }
    }

    private void handle(InputStream input, OutputStream output) throws IOException {
        // 为了方便操作流，使用缓存字符流对字节流进行封装：
        BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        BufferedReader reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        // 服务器端首先向客户端发送一条问候消息：
        writer.write("hello\n");
        writer.flush();
        for (; ; ) {
            // 之后服务器端不断从客户端接收消息。程序中所使用的流都是IO包中的流，属于阻塞IO：
            String s = reader.readLine();
            if ("bye".equals(s)) {
                writer.write("bye\n");
                writer.flush();
                break;
            }
            writer.write("ok: " + s + "\n");
            writer.flush();
        }
    }
}
```

### 客户端

```Java
public class Client {
    public static void main(String[] args) throws IOException {
        // 连接到服务器端，如果连接成功，将返回一个Socket实例，用于后续通信：
        Socket sock = new Socket("127.0.0.1", 6666);
        try (InputStream input = sock.getInputStream()) {
            try (OutputStream output = sock.getOutputStream()) {
                Client.handle(input, output);
            }
        }
        sock.close();
        System.out.println("disconnected.");
    }

    private static void handle(InputStream input, OutputStream output) throws IOException {
        BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        BufferedReader reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        Scanner scanner = new Scanner(System.in);
        System.out.println("[server] " + reader.readLine());
        for (; ; ) {
            System.out.print(">>> ");
            String s = scanner.nextLine();
            // 客户端向服务器端发送读取的输入内容：
            writer.write(s);
            writer.newLine();
            writer.flush();
            // 客户端读取从服务器端发回的响应内容：
            String resp = reader.readLine();
            System.out.println("<<< " + resp);
            if ("bye".equals(resp)) {
                break;
            }
        }
    }
}
```

### Socket流

当Socket连接创建成功后，无论是服务器端，还是客户端，我们都使用Socket实例进行网络通信。因为TCP是一种基于流的协议，因此，Java标准库使用InputStream和OutputStream来封装Socket的数据流，这样我们使用Socket的流，和普通IO流类似：

```Java
// 用于读取网络数据:
InputStream in = sock.getInputStream();
// 用于写入网络数据:
OutputStream out = sock.getOutputStream();
```

如果不调用flush()，我们很可能会发现，客户端和服务器都收不到数据，这并不是Java标准库的设计问题，而是我们以流的形式写入数据的时候，并不是一写入就立刻发送到网络，而是先写入内存缓冲区，直到缓冲区满了以后，才会一次性真正发送到网络，这样设计的目的是为了提高传输效率。如果缓冲区的数据很少，而我们又想强制把这些数据发送到网络，就必须调用flush()强制把缓冲区数据发送出去。

### 小结

使用Java进行TCP编程时，需要使用Socket模型：

- 服务器端用ServerSocket监听指定端口；
- 客户端使用Socket(InetAddress, port)连接服务器；
- 服务器端用accept()接收连接并返回Socket；
- 双方通过Socket打开InputStream/OutputStream读写数据；
- 服务器端通常使用多线程同时处理多个客户端连接，利用线程池可大幅提升效率；
- flush()用于强制输出缓冲区到网络。

## UDP编程

和TCP编程相比，UDP编程就简单得多，因为UDP没有创建连接，数据包也是一次收发一个，所以没有流的概念。

在Java中使用UDP编程，仍然需要使用Socket，因为应用程序在使用UDP时必须指定网络接口（IP）和端口号。注意：UDP端口和TCP端口虽然都使用0-65535，但他们是两套独立的端口，即一个应用程序用TCP占用了端口1234，不影响另一个应用程序用UDP占用端口1234。

### 服务器端

### 客户端

### 小结
