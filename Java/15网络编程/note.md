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

```Java
public class Server {
    public static void main(String[] args) throws IOException {
        // 使用UDP也需要监听指定的端口：
        DatagramSocket ds = new DatagramSocket(6666);
        System.out.println("server is running...");
        for (; ; ) {
            // 准备一个缓冲区：
            byte[] buffer = new byte[1024];
            // 通过DatagramPacket实现接收：
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            // 收取一个UDP数据包：
            ds.receive(packet);
            // 收取的数据被存储在buffer中，按照起始位置、长度、编码将byte数组转换为String：
            // String​(byte[] bytes, int offset, int length, Charset charset)
            String cmd = new String(packet.getData(), packet.getOffset(), packet.getLength(), StandardCharsets.UTF_8);
            String resp = "bad command";
            switch (cmd) {
                case "date":
                    resp = LocalDate.now().toString();
                    break;
                case "time":
                    resp = LocalTime.now().withNano(0).toString();
                    break;
                case "datetime":
                    resp = LocalDateTime.now().withNano(0).toString();
                    break;
                case "weather":
                    resp = "sunny";
                    break;
                default:
                    break;
            }
            System.out.println(cmd + " >>> " + resp);
            packet.setData(resp.getBytes(StandardCharsets.UTF_8));
            // 发送一个UDP数据包：
            ds.send(packet);
        }
    }
}
```

当服务器收到一个DatagramPacket后，通常必须立刻回复一个或多个UDP包，因为客户端地址在DatagramPacket中，每次收到的DatagramPacket可能是不同的客户端，如果不回复，客户端就收不到任何UDP包。

### 客户端

```Java
public class Client {
    public static void main(String[] args) throws IOException, InterruptedException {
        // 客户端创建DatagramSocket实例时并不需要指定端口，而是由操作系统自动指定一个当前未使用的端口：
        DatagramSocket ds = new DatagramSocket();
        // 调用setSoTimeout(1000)设定超时1秒，意思是后续接收UDP包时，等待时间最多不会超过1秒，否则在没有收到UDP包时，客户端会无限等待下去：
        // 这一点和服务器端不一样，服务器端可以无限等待，因为它本来就被设计成长时间运行：
        ds.setSoTimeout(1000);
        // 这个connect()方法不是真连接，它是为了在客户端的DatagramSocket实例中保存服务器端的IP和端口号，确保这个DatagramSocket实例只能往指定的地址和端口发送UDP包，不能往其它地址和端口发送：
        // 这么做不是UDP的限制，而是Java内置了安全检查：
        ds.connect(InetAddress.getByName("localhost"), 6666);
        DatagramPacket packet = null;
        for (int i = 0; i < 5; i++) {
            // 发送：
            String cmd = new String[]{"date", "time", "datetime", "weather", "hello"}[i];
            byte[] data = cmd.getBytes();
            packet = new DatagramPacket(data, data.length);
            ds.send(packet);
            // 接收：
            byte[] buffer = new byte[1024];
            packet = new DatagramPacket(buffer, buffer.length);
            ds.receive(packet);
            String resp = new String(packet.getData(), packet.getOffset(), packet.getLength());
            System.out.println(cmd + " >>> " + resp);
            Thread.sleep(1500);
        }
        // 注意到disconnect()也不是真正地断开连接，它只是清除了客户端DatagramSocket实例记录的远程服务器地址和端口号，这样，DatagramSocket实例就可以连接另一个服务器端：
        ds.disconnect();
        System.out.println("disconnected.");
    }
}
```

如果客户端希望向两个不同的服务器发送UDP包，那么它必须创建两个DatagramSocket实例。

后续的收发数据和服务器端是一致的。通常来说，客户端必须先发UDP包，因为客户端不发UDP包，服务器端就根本不知道客户端的地址和端口号。

### 小结

使用UDP协议通信时，服务器和客户端双方无需建立连接：

- 服务器端用DatagramSocket(port)监听端口；
- 客户端使用DatagramSocket.connect()指定远程地址和端口；
- 双方通过receive()和send()读写数据；
- **DatagramSocket没有IO流接口，数据被直接写入byte[]缓冲区**。

## HTTP编程

HTTP是HyperText Transfer Protocol的缩写，翻译为超文本传输协议，它是基于TCP协议之上的一种请求-响应协议。

当浏览器希望访问某个网站时，浏览器和网站服务器之间首先建立TCP连接，且服务器总是使用80端口和加密端口443，然后，浏览器向服务器发送一个HTTP请求，服务器收到后，返回一个HTTP响应，并且在响应中包含了HTML的网页内容，这样，浏览器解析HTML后就可以给用户显示网页了。

**POST请求通常要设置Content-Type表示Body的类型，Content-Length表示Body的长度，这样服务器就可以根据请求的Header和Body做出正确的响应**。POST请求的参数不一定是URL编码，可以按任意格式编码，只需要在Content-Type中正确设置即可。

对于最早期的HTTP/1.0协议，每次发送一个HTTP请求，客户端都需要先创建一个新的TCP连接，然后，收到服务器响应后，关闭这个TCP连接。由于建立TCP连接就比较耗时，因此，为了提高效率，HTTP/1.1协议允许在一个TCP连接中反复发送-响应，这样就能大大提高效率。

因为HTTP协议是一个请求-响应协议，客户端在发送了一个HTTP请求后，必须等待服务器响应后，才能发送下一个请求，这样一来，如果某个响应太慢，它就会堵住后面的请求。所以，为了进一步提速，HTTP/2.0允许客户端在没有收到响应的时候，发送多个HTTP请求，服务器返回响应的时候，不一定按顺序返回，只要双方能识别出哪个响应对应哪个请求，就可以做到并行发送和接收。

### HTTP编程

因为浏览器也是一种HTTP客户端，所以，客户端的HTTP编程，它的行为本质上和浏览器是一样的，即发送一个HTTP请求，接收服务器响应后，获得响应内容。只不过浏览器进一步把响应内容解析后渲染并展示给了用户，而我们使用Java进行HTTP客户端编程仅限于获得响应内容。

```Java
public class Main {
    // 创建一个全局HttpClient实例，因为HttpClient内部使用线程池优化多个HTTP连接，可以复用：
    static HttpClient httpClient = HttpClient.newBuilder().build();

    public static void main(String[] args) throws Exception {
        httpGet("https://www.sina.com.cn/");
        httpPost("https://accounts.douban.com/j/mobile/login/basic",
                "name=bob%40example.com&password=12345678&remember=false&ck=&ticket=");
        httpGetImage("https://img.t.sinajs.cn/t6/style/images/global_nav/WB_logo.png");
    }
    // 使用GET请求获取文本内容：
    static void httpGet(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(new URI(url))
                .header("User-Agent", "Java HttpClient").header("Accept", "*/*")
                .timeout(Duration.ofSeconds(5))
                .version(HttpClient.Version.HTTP_2).build();
        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println(response.body().substring(0, 1024) + "...");
    }

    static void httpGetImage(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(new URI(url))
                .header("User-Agent", "Java HttpClient").header("Accept", "*/*")
                .timeout(Duration.ofSeconds(5))
                .version(HttpClient.Version.HTTP_2).build();
        HttpResponse<InputStream> response = httpClient.send(request, HttpResponse.BodyHandlers.ofInputStream());
        BufferedImage img = ImageIO.read(response.body());
        ImageIcon icon = new ImageIcon(img);
        JFrame frame = new JFrame();
        frame.setLayout(new FlowLayout());
        frame.setSize(200, 100);
        JLabel lbl = new JLabel();
        lbl.setIcon(icon);
        frame.add(lbl);
        frame.setVisible(true);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
    }

    static void httpPost(String url, String body) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(new URI(url))
                .header("User-Agent", "Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0) like Gecko")
                .header("Accept", "*/*").header("Content-Type", "application/x-www-form-urlencoded")
                .timeout(Duration.ofSeconds(5))
                .version(HttpClient.Version.HTTP_2)
                .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8)).build();
        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println(response.body());
    }
}
```

### 小结
