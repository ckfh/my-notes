# 开发Web应用

直接使用Servlet进行Web开发好比直接在JDBC上操作数据库，比较繁琐，更好的方法是在Servlet基础上封装MVC框架，基于MVC开发Web应用，大部分时候，不需要接触Servlet API，开发省时省力。

## 使用Spring MVC

Java Web的基础：Servlet容器，以及标准的Servlet组件：

- Servlet：能处理HTTP请求并将HTTP响应返回；
- JSP：一种嵌套Java代码的HTML，将被编译为Servlet；
- Filter：能过滤指定的URL以实现拦截功能；
- Listener：监听指定的事件，如ServletContext、HttpSession的创建和销毁。

此外，Servlet容器为每个Web应用程序自动创建一个唯一的ServletContext实例，这个实例就代表了Web应用程序本身。

但是，Spring提供的是一个IoC容器，所有的Bean，包括Controller，都在Spring IoC容器中被初始化，而Servlet容器由JavaEE服务器提供（如Tomcat），Servlet容器对Spring一无所知，他们之间到底依靠什么进行联系，又是以何种顺序初始化的？

现在是Web应用程序，而Web应用程序总是由Servlet容器创建，那么，Spring容器应该由谁创建？在什么时候创建？Spring容器中的Controller又是如何通过Servlet调用的？

早期的Servlet需要在web.xml中配置映射路径，但最新Servlet版本只需要通过注解就可以完成映射。

我们在web.xml中配置Spring MVC提供的DispatcherServlet：

```XML
<!DOCTYPE web-app PUBLIC
        "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
    <display-name>Archetype Created Web Application</display-name>

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
        </init-param>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.cat.AppConfig</param-value>
        </init-param>
        <load-on-startup>0</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```

初始化参数contextClass指定使用注解配置的AnnotationConfigWebApplicationContext，配置文件的位置参数contextConfigLocation指向AppConfig的完整类名，最后，把这个Servlet映射到/*，即处理所有URL。

上述配置可以看作一个样板配置，有了这个配置，Servlet容器会首先初始化Spring MVC的DispatcherServlet，在DispatcherServlet启动时，它根据配置AppConfig创建了一个类型是WebApplicationContext的IoC容器，完成所有Bean的初始化，并将容器绑到ServletContext上。

因为DispatcherServlet持有IoC容器，能从IoC容器中获取所有@Controller的Bean，因此，DispatcherServlet接收到所有HTTP请求后，根据Controller方法配置的路径，就可以正确地把请求转发到指定方法，并根据返回的ModelAndView决定如何渲染页面。

使用Spring MVC时，整个Web应用程序按如下顺序启动：

  1. 启动Tomcat服务器；
  2. Tomcat读取web.xml并初始化DispatcherServlet；
  3. DispatcherServlet创建IoC容器并自动注册到ServletContext中。

启动后，浏览器发出的HTTP请求全部由DispatcherServlet接收，并根据配置转发到指定Controller的指定方法处理。

编写Controller只需要遵循以下要点：

- Controller必须标记为@Controller；
- 一个方法对应一个HTTP请求路径，用@GetMapping或@PostMapping表示GET或POST请求；
- 需要接收的HTTP参数以@RequestParam()标注，可以设置默认值。如果方法参数需要传入HttpServletRequest、HttpServletResponse或者HttpSession，直接添加这个类型的参数即可，Spring MVC会自动按类型传入；
- 返回的ModelAndView通常包含View的路径和一个Map作为Model，但也可以没有Model；
- 返回重定向时既可以写new ModelAndView("redirect:/signin")，也可以直接返回"redirect:/signin"；
- **如果在方法内部直接操作HttpServletResponse发送响应，返回null表示无需进一步处理**；

    ```Java
    public ModelAndView download(HttpServletResponse response) {
        byte[] data = ...
        response.setContentType("application/octet-stream");
        OutputStream output = response.getOutputStream();
        output.write(data);
        output.flush();
        return null;
    }
    ```

- 对URL进行分组，每组对应一个Controller是一种很好的组织形式，并可以在Controller的class定义出添加URL前缀；

    ```Java
    @Controller
    @RequestMapping("/user")
    public class UserController {
        // 注意实际URL映射是/user/profile
        @GetMapping("/profile")
        public ModelAndView profile() {
            ...
        }

        // 注意实际URL映射是/user/changePassword
        @GetMapping("/changePassword")
        public ModelAndView changePassword() {
            ...
        }
    }
    ```

- **实际方法的URL映射总是前缀+路径，这种形式还可以有效避免不小心导致的重复的URL映射**。
