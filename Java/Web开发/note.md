# 开发

## Web基础

```Java
public class HTTPServer {
    public static void main(String[] args) throws IOException {
        // 一个简单的HTTP服务器，本质就是一个TCP服务器
        ServerSocket ss = new ServerSocket(8080); // 监听8080端口
        System.out.println("server is running...");
        for (; ; ) {
            Socket sock = ss.accept(); // 接收到一个socket对象，表示有客户端发起链接，通过该对象进行通信
            System.out.println("connected from " + sock.getRemoteSocketAddress());
            Thread t = new HTTPHandler(sock); // 每接收一个socket对象，就启用一个线程进行处理
            t.start();
        }
    }
}

class HTTPHandler extends Thread {
    Socket sock;

    public HTTPHandler(Socket sock) {
        this.sock = sock;
    }

    @Override
    public void run() {
        // 获取socket对象的输入流和输出流，即HTTP请求和HTTP响应，传递给处理函数
        try (InputStream input = this.sock.getInputStream()) {
            try (OutputStream output = this.sock.getOutputStream()) {
                this.handle(input, output);
            }
        } catch (Exception e) {
            try {
                this.sock.close(); // 关闭socket对象的同时，输入输出流也被关闭
            } catch (IOException ioe) {
                ioe.printStackTrace();
            }
            System.out.println("client disconnected.");
        }
    }

    private void handle(InputStream input, OutputStream output) throws IOException {
        System.out.println("Process new http request...");
        // 将字节流转换为缓冲字符流，方便进行处理
        BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        BufferedReader reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        boolean requestOk = false;
        String first = reader.readLine(); // 读取请求消息的第一行即请求行
        if (first.startsWith("GET / HTTP/1.")) {
            // GET方法访问，URI即访问目标为路径/，HTTP版本为1.0或1.1
            requestOk = true;
        }
        for (; ; ) {
            String header = reader.readLine(); // 读取请求消息中的消息头，一直读取到空行为止
            if (header.isEmpty())
                break;
            System.out.println(header);
        }
        // 根据请求行确定响应消息
        System.out.println(requestOk ? "Response OK" : "Response Error");
        if (!requestOk) {
            writer.write("404 Not Found\r\n"); // 状态行
            writer.write("Content-Length: 0\r\n"); // 消息头
            writer.write("\r\n"); // 空行之后是消息体，无内容
            writer.flush();
        } else {
            String data = "<html><body><h1>Hello, World!</h1></body></html>"; // 消息体内容
            int length = data.getBytes(StandardCharsets.UTF_8).length;
            writer.write("HTTP/1.0 200 OK\r\n"); // 状态行，HTTP版本/状态码/响应短语
            writer.write("Connection: close\r\n"); // 消息头一
            writer.write("Content-Type: text/html\r\n"); // 消息头二
            writer.write("Content-Length: " + length + "\r\n"); // 消息头三
            writer.write("\r\n"); // 空行
            writer.write(data); // 消息体
            writer.flush();
        }
    }
}
```

![页面](.\image\页面.jpg)

## Servlet入门

- 在JavaEE平台上，处理TCP连接，解析HTTP协议这些底层工作统统扔给现成的Web服务器去做，我们只需要把自己的应用程序跑在Web服务器上。为了实现这一目的，JavaEE提供了Servlet API，我们使用Servlet API编写自己的Servlet来处理HTTP请求，Web服务器实现Servlet API接口，实现底层功能。  
    ![过程](.\image\过程.jpg)
- 编写Web应用程序就是编写Servlet处理HTTP请求；Servlet API提供了HttpServletRequest和HttpServletResponse两个高级接口来封装HTTP请求和响应；Web应用程序必须按固定结构组织并打包为.war文件；需要启动Web服务器来加载我们的war包来运行Servlet。
- 一个Servlet总是继承自HttpServlet，然后覆写doGet()或doPost()方法。注意到doGet()方法传入了HttpServletRequest和HttpServletResponse两个对象，分别代表HTTP请求和响应。我们使用Servlet API时，并不直接与底层TCP交互，也不需要解析HTTP协议，因为HttpServletRequest和HttpServletResponse就已经封装好了请求和响应。以发送响应为例，我们只需要设置正确的响应类型，然后获取PrintWriter，写入响应即可。

    ```Java
    @WebServlet(urlPatterns = "/") // WebServlet注解表示这是一个Servlet，并映射到地址/
    public class HelloServlet extends HttpServlet {
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            LocalTime t = LocalTime.now();
            resp.setContentType("text/html"); // 设置响应类型
            PrintWriter pw = resp.getWriter(); // 获取输出流
            pw.println("<h1>Hello, World!</h1>"); // 写入响应
            pw.print("<h1>" + t + "</h1>");
            pw.flush(); // 强制输出
        }
    }
    ```

    ![servlet页面](.\image\servlet页面.jpg)

- **为啥路径是/hello/而不是/。因为一个Web服务器允许同时运行多个Web App，而我们的Web App叫hello，因此，第一级目录/hello表示Web App的名字，后面的/才是我们在HelloServlet中映射的路径。**
- 类似Tomcat这样的服务器也是Java编写的，启动Tomcat服务器实际上是启动Java虚拟机，执行Tomcat的main()方法，然后由Tomcat负责加载我们的.war文件，并创建一个HelloServlet实例，最后以多线程的模式来处理HTTP请求。如果Tomcat服务器收到的请求路径是/（假定部署文件为ROOT.war），就转发到HelloServlet并传入HttpServletRequest和HttpServletResponse两个对象。
- 因为我们编写的Servlet并不是直接运行，而是由Web服务器加载后创建实例运行，所以，类似Tomcat这样的Web服务器也称为Servlet容器。
- 在Servlet容器中运行的Servlet具有如下特点：无法在代码中直接通过new创建Servlet实例，必须由Servlet容器自动创建Servlet实例；Servlet容器只会给每个Servlet类创建**唯一实例**；Servlet容器会使用**多线程**执行doGet()或doPost()方法。
- 因此，在Servlet中定义的**实例变量**会被多个线程同时访问，要注意线程安全；HttpServletRequest和HttpServletResponse实例是由Servlet容器传入的局部变量，它们只能被当前线程访问，不存在多个线程访问的问题；在doGet()或doPost()方法中，如果使用了ThreadLocal，但没有清理，那么它的状态很可能会影响到下次的某个请求，因为Servlet容器很可能用线程池实现线程复用。
- 正确编写Servlet，要清晰理解Java的多线程模型，需要同步访问的必须同步。

### 小结

> 使用idea新建一个maven web项目，新建后删除webapp文件夹中的index.jsp文件，在WEB-INF目录中放置web.xml文件，在webapp同级目录中新建java和resources文件夹。  
> 引入servlet-api依赖，注意api版本和之后部署的tomcat版本需要对应，例如tomcat9支持的是servlet 4.0。  
> 编写servlet程序映射指定路径，在doGet()或doPost()方法中进行请求和响应的处理。  
> 使用mvn clean package将项目打包为war包，放置在tomcat安装目录中的webapps文件夹中，切换到bin目录，使用startup.bat启动tomcat服务器，使用shutdown.bat关闭tomcat服务器。

## Servlet开发

- Tomcat实际上也是一个Java程序，我们看看Tomcat的启动流程：启动JVM并执行Tomcat的main()方法；加载war并初始化Servlet；正常服务。
- 启动Tomcat无非就是设置好classpath并执行Tomcat某个jar包的main()方法，我们完全可以把Tomcat的jar包全部引入进来，然后自己编写一个main()方法，先启动Tomcat，然后让它加载我们的webapp就行。
- 引入依赖tomcat-embed-core和tomcat-embed-jasper，不必引入Servlet API，因为引入Tomcat依赖后自动引入了Servlet API。

```Java
public class Main {
    public static void main(String[] args) throws Exception {
        // 启动Tomcat:
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(Integer.getInteger("port", 8080));
        tomcat.getConnector();
        // 创建webapp:
        Context ctx = tomcat.addWebapp("", new File("src/main/webapp").getAbsolutePath());
        WebResourceRoot resources = new StandardRoot(ctx);
        resources.addPreResources(
                new DirResourceSet(resources, "/WEB-INF/classes", new File("target/classes").getAbsolutePath(), "/"));
        ctx.setResources(resources);
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

- 直接运行main()方法，即可启动嵌入式Tomcat服务器，然后，通过预设的tomcat.addWebapp("", new File("src/main/webapp")，Tomcat会自动加载当前工程作为**根webapp**。
- 通过main()方法启动Tomcat服务器并加载我们自己的webapp有如下好处：启动简单，无需下载Tomcat或安装任何IDE插件；调试方便，可在IDE中使用断点调试；使用Maven创建war包后，也可以正常部署到独立的Tomcat服务器中。
- 开发Servlet时，推荐使用main()方法启动嵌入式Tomcat服务器并加载当前工程的webapp，便于开发调试，且不影响打包部署，能极大地提升开发效率。

![方案](.\image\方案.jpg)

## Servlet进阶

- 早期的Servlet需要在web.xml中配置映射路径，但最新Servlet版本只需要通过注解就可以完成映射。
- 一个Webapp中的多个Servlet依靠路径映射来处理不同的请求；
- 映射为/的Servlet可处理所有“未匹配”的请求；
- 如何处理请求取决于Servlet覆写的对应方法；
- Web服务器通过多线程处理HTTP请求，一个Servlet的处理方法可以由多线程并发执行。
- 一个Servlet类在服务器中只有一个实例，但对于每个HTTP请求，Web服务器会使用多线程执行请求。因此，一个Servlet的doGet()、doPost()等处理请求的方法是多线程并发执行的。**如果Servlet中定义了字段**，要注意多线程并发访问的问题。
- 对于每个请求，Web服务器会创建唯一的HttpServletRequest和HttpServletResponse实例，因此，HttpServletRequest和HttpServletResponse实例只有在当前处理线程中有效，它们总是局部变量，不存在多线程共享的问题。

### 重定向与转发

#### Redirect

- 重定向是指当浏览器请求一个URL时，服务器返回一个重定向指令，告诉浏览器地址已经变了，麻烦使用新的URL再重新发送新请求。

    ```Java
    @WebServlet(urlPatterns = "/hi")
    public class RedirectServlet extends HttpServlet {
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            // 构造重定向路径
            String name = req.getParameter("name");
            String redirectToUrl = "/hello" + (name == null ? "" : "?name=" + name);
            // 如果浏览器发送 GET /hi 请求，将发送一个重定向响应
            resp.sendRedirect(redirectToUrl); // 临时重定向
        }
    }
    ```

    ![重定向响应](.\image\重定向响应.jpg)

- 当浏览器收到302响应后，它会立刻根据Location的指示发送一个新的GET /hello请求，这个过程就是重定向。

    ![重定向过程](.\image\重定向过程.jpg)

- 可以观察到浏览器发送了两次HTTP请求，并且浏览器的地址栏路径自动更新为/hello。

    ![传参重定向](.\image\传参重定向.jpg)

- 重定向有两种：一种是302响应，称为临时重定向，一种是301响应，称为永久重定向。两者的区别是，如果服务器发送301永久重定向响应，浏览器会**缓存**/hi到/hello这个重定向的关联，下次请求/hi的时候，浏览器就直接发送/hello请求了。

    ```Java
    // 实现301永久重定向的做法
    resp.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY);
    resp.setHeader("Location", redirectToUrl);
    ```

- 当使用301永久重定向时的几个注意点：1.第一次访问的路径会被缓存，之后访问时在chrome浏览器中会显示该重定向来自缓存。2.启用301永久重定向时的所有访问路径都会被缓存，如果之后切换为302临时重定向，此时被缓存的路径仍然以301响应码进行返回，新的路径则以302响应码进行返回。3.如果想要禁止路径缓存，可以在chrome控制台中选择disable cache。

    ![永久重定向](.\image\永久重定向.jpg)

    ![永久重定向缓存](.\image\永久重定向缓存.jpg)

- **重定向的目的是当Web应用升级后，如果请求路径发生了变化，可以将原来的路径重定向到新路径，从而避免浏览器请求原路径找不到资源。**

#### Forward

- Forward是指内部转发。当一个Servlet处理请求的时候，它可以决定自己不继续处理，而是转发给另一个Servlet处理。
- **转发和重定向的区别在于，转发是在Web服务器内部完成的，对浏览器来说，它只发出了一个HTTP请求。**

    ![转发](.\image\转发.jpg)

    ```Java
    @WebServlet(urlPatterns = "/morning")
    public class ForwardServlet extends HttpServlet {
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            // 把请求和响应都转发给路径为/hello的Servlet。
            req.getRequestDispatcher("/hello").forward(req, resp);
        }
    }
    ```

- 注意到使用转发的时候，浏览器的地址栏路径仍然是/morning，浏览器并不知道该请求在Web服务器内部实际上做了一次转发。

### 使用Session和Cookie

- 在Web应用程序中，我们经常要跟踪用户身份。当一个用户登录成功后，如果他继续访问其它页面，Web程序如何才能识别出该用户身份。
- 因为HTTP协议是一个无状态协议，即Web应用程序无法区分收到的两个HTTP请求是否是同一个浏览器发出的。为了跟踪用户状态，服务器可以向浏览器分配一个唯一ID，并以Cookie的**形式**发送到浏览器，浏览器在后续访问时总是附带此Cookie，这样，服务器就可以识别用户身份。
- 我们把这种基于唯一ID识别用户身份的**机制**称为Session。每个用户第一次访问服务器后，会自动获得一个Session ID。如果用户在一段时间内没有访问服务器，那么Session会自动失效，下次即使带着上次分配的Session ID访问，服务器也认为这是一个新用户，会分配新的Session ID。
- stackoverflow上有关session和cookie关系的回答：[参考链接](https://stackoverflow.com/questions/32563236/relation-between-sessions-and-cookies)
